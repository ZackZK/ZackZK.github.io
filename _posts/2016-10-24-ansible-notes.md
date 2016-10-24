---
layout: post
title: ansible使用问题总结
description: 
modified: 2016-10-24
tags: [ansible]

---

## 1. Network is unreachable问题
问题： 使用apt-get在目标主机可正常安装，但是通过ansible的apt模块报 Network is unreachable的问题
原因： 目标主机上访问外网需要设置代理， ansible默认不会执行目标主机的.bashrc设置环境变量
解决方法： 使用ansible的environment关键字来设置proxy
见官方文档： http://docs.ansible.com/ansible/playbooks_environment.html

官方的例子

{% highlight bash%}
- hosts: all
  remote_user: root

  tasks:

    - apt: name=apache2 state=installed
      environment:
      http_proxy: http://proxy.example.com:8080
      
{% endhighlight %}

也可以使用变量：

{% highlight bash%}
- hosts: all
  remote_user: root

  # here we make a variable named "proxy_env" that is a dictionary
  vars:
    proxy_env:
      http_proxy: http://proxy.example.com:8080

  tasks:

    - apt: name=apache2 state=installed
      environment: "{{proxy_env}}"
{% endhighlight %}
