---
layout: post
title: spark 读取hbase及在集群中的相关配置
description: "介绍spark的读取hbase及在集群中的配置"
modified: 2015-09-12
tags: [ spark, hbase, spark cluster, spark configuration]
---

对于读取hbase，spark提供了newAPIHadoopRDD接口可以很方便的读取hbase内容。下面是一个具体的例子：

首先，加入下面依赖：

{% highlight xml %}
    <dependencies>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.10</artifactId>
            <version>1.4.1</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-common</artifactId>
            <version>1.1.2</version>
        </dependency>

        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-client</artifactId>
            <version>1.1.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-server</artifactId>
            <version>1.1.2</version>
        </dependency>
    </dependencies>
{% endhighlight %}

下面的例子从hbase数据库中读取数据，并进行RDD操作，生成两两组合。

{% highlight java %}

import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.util.Base64;
import org.apache.hadoop.hbase.util.Bytes;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.function.Function;
import org.apache.spark.api.java.function.Function2;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.client.Scan;
import org.apache.hadoop.hbase.client.Result;
import org.apache.hadoop.hbase.io.ImmutableBytesWritable;
import org.apache.hadoop.hbase.mapreduce.TableInputFormat;
import org.apache.spark.api.java.JavaPairRDD;

import org.apache.hadoop.hbase.protobuf.ProtobufUtil;
import org.apache.hadoop.hbase.protobuf.generated.ClientProtos;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple10;
import scala.Tuple2;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

/*
* hbase content:
* hbase(main):003:0> scan 'scores'
ROW                             COLUMN+CELL
 Jim                            column=course:art, timestamp=1441549968185, value=80
 Jim                            column=course:math, timestamp=1441549968167, value=89
 Jim                            column=grade:, timestamp=1441549968142, value=4
 Tom                            column=course:art, timestamp=1441549912222, value=88
 Tom                            column=course:math, timestamp=1441549842348, value=97
 Tom                            column=grade:, timestamp=1441549820516, value=5
2 row(s) in 0.1330 seconds
*/
public class spark_hbase_main {
    private static String appName = "Hello";

    public static void main(String[] args) {
        SparkConf sparkConf = new SparkConf().setAppName(appName);
        JavaSparkContext jsc = new JavaSparkContext(sparkConf);

        Configuration conf = HBaseConfiguration.create();

        Scan scan = new Scan();
        scan.addFamily(Bytes.toBytes("course"));
        scan.addColumn(Bytes.toBytes("course"), Bytes.toBytes("art"));
        scan.addColumn(Bytes.toBytes("course"), Bytes.toBytes("math"));

        String scanToString = "";
        try {
            ClientProtos.Scan proto = ProtobufUtil.toScan(scan);
            scanToString = Base64.encodeBytes(proto.toByteArray());
        } catch (IOException io) {
            System.out.println(io);
        }

        for (int i = 0; i < 2; i++) {
            try {
                String tableName = "scores";
                conf.set(TableInputFormat.INPUT_TABLE, tableName);
                conf.set(TableInputFormat.SCAN, scanToString);

                //获得hbase查询结果Result
                JavaPairRDD<ImmutableBytesWritable, Result> hBaseRDD = jsc.newAPIHadoopRDD(conf,
                        TableInputFormat.class, ImmutableBytesWritable.class,
                        Result.class);

                /* 生成类似 [(Jim, 80, 89), (Tom, 88, 97)] 的RDD */
                JavaPairRDD<String, List<Integer>> art_scores = hBaseRDD.mapToPair(
                        new PairFunction<Tuple2<ImmutableBytesWritable, Result>, String, List<Integer>>() {
                            @Override
                            public Tuple2<String, List<Integer>> call(Tuple2<ImmutableBytesWritable, Result> results) {

                                List<Integer> list = new ArrayList<Integer>();

                                byte[] art_score = results._2().getValue(Bytes.toBytes("course"), Bytes.toBytes("art"));
                                byte[] math_score = results._2().getValue(Bytes.toBytes("course"), Bytes.toBytes("math"));

                                /* 注意： Hbase里存的数据以Byte Array形式存储， 需要使用Integer.parseInt(Bytes.toString(art_score))将数据内容转化为整型
                                * Integer.parseInt(price.toString()) 会得到错误答案 */
                                list.add(Integer.parseInt(Bytes.toString(art_score)));
                                list.add(Integer.parseInt(Bytes.toString(math_score)));

                                return new Tuple2<String, List<Integer>>(Bytes.toString(results._1().get()), list);
                            }
                        }
                );

                /* 如果是使用Java8， 可以简化成下面的形式: 
                JavaPairRDD<Integer, Double> stock_price_pair = hBaseRDD.mapToPair(
                    (results) -> {
                          List<Integer> list = new ArrayList<Integer>();

                          byte[] art_score = results._2().getValue(Bytes.toBytes("course"), Bytes.toBytes("art"));
                          byte[] math_score = results._2().getValue(Bytes.toBytes("course"), Bytes.toBytes("math"));

                          list.add(Integer.parseInt(Bytes.toString(art_score)));
                          list.add(Integer.parseInt(Bytes.toString(math_score)));

                          return new Tuple2<String, List<Integer>>(Bytes.toString(results._1().get()), list);
                    }
                )
                */

                /* 笛卡尔乘积，生成 [((Jim, 80, 89), (Tom, 88, 97)), ((Tom, 88, 97), (Jim, 80, 89)), ((Jim, 80, 89), (Jim, 80, 89)),
                ((Tom, 88, 97), (Tom, 88, 97))] 的RDD */] */
                JavaPairRDD<Tuple2<String, List<Integer>>, Tuple2<String, List<Integer>>> cart = art_scores.cartesian(art_scores);

                /* 利用row key的大小关系去除重复的组合关系， 生成 [((Jim, 80, 89), (Tom, 88, 97))] */
                JavaPairRDD<Tuple2<String, List<Integer>>, Tuple2<String, List<Integer>>> cart2 = cart.filter(
                        new Function<Tuple2<Tuple2<String, List<Integer>>, Tuple2<String, List<Integer>>>, Boolean>() {
                            public Boolean call(Tuple2<Tuple2<String, List<Integer>>, Tuple2<String, List<Integer>>> tuple2Tuple2Tuple2) throws Exception {

                                return tuple2Tuple2Tuple2._1()._1().compareTo(tuple2Tuple2Tuple2._2()._1()) < 0;
                            }
                        }
                );

                /* 得到最终结果 */
                cart_all = cart2.collect();

            } catch (Exception e) {
                System.out.println(e);
            }
        }
    }
}
{% endhighlight %}
