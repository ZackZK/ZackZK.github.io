---
layout: post
title: spark 配置及使用
description: "介绍spark的相关配置和使用"
modified: 2015-09-09
tags: [ spark, spark configuration ]
---

## spark配置 ##
spark的配置文件放在conf目录下，默认会有类似xx.template的模板文件，去掉后缀.template就可以作为spark相应的配置文件。

### spark UI的IP和port口配置 ###
比如，UI的ip和port口配置，在云服务器上，比如阿里云主机上，eth0默认是内部网络ip（用于阿里云主机之间通信），eth1才是外网可访问的ip地址。
spark会使用eth0的ip地址作为UI 的ip地址，在conf/spark-env.sh文件中加入下面内容即可

{% highlight bash %}
export SPARK_MASTER_IP=12.12.12.12
export SPARK_MASTER_WEBUI_PORT=8080
{% endhighlight %}

注意： 如果是在云环境下，ip地址指定为外部网络

### Spark的log级别设置 ###
默认情况下，spark会在屏幕打印INFO级别以上的log，这些信息对于调试来说很重要，但是当spark程序正常运行时，可能不需要这么多信息，这个时候，可以通过修改
conf/log4j.properties：
在文件中找到该行：
{% highlight bash %}
log4j.rootCategory=INFO, console
{% endhighlight %}
将其修改成
{% highlight bash %}
log4j.rootCategory=WARN, console
{% endhighlight %}


