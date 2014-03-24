---
layout: post
title: "HttpStubStatusModule源码分析"
description: "HttpStubStatusModule源码分析"
category: "NGINX源码分析"
tags: ["direct io", "aio", "NGINX源码分析"]
---
{% include JB/setup %}

##Synopsis##
这个模块主要用来获取nginx当前执行状况，而且默认是未被编译进nginx中的。  
```--with-http_stub_status_module```

Example:
{% highlight bash %}
location /nginx_status {
  stub_status on;
  access_log   off;
  allow SOME.IP.ADD.RESS;
  deny all;
}
{% endhighlight %}

{% highlight bash %}
[root@localhost sbin]# curl http://localhost/nginx_status
Active connections: 1 
server accepts handled requests
 9 9 9 
Reading: 0 Writing: 1 Waiting: 0 
{% endhighlight %}

##Source Code##
代码在ngx_http_sub_status_module.c 如果没有变量，这简直就是个hello world。如果看不懂，[Nginx模块开发入门] [1]。
关于变量部分另外再说吧。



[1]: http://blog.codinglabs.org/articles/intro-of-nginx-module-development.html
[2]: http://wiki.nginx.org/HttpStubStatusModule