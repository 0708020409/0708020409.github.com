---
layout: post
title: "常用非标准HTTP Header"
description: ""
category: "HTTP"
tags: ["NGINX", "HTTP"]
---
{% include JB/setup %}

##real ip##
  在实际的应用中，通常需要取得客户端的ip进行处理，但是假如应用和客户端之间有台nginx服务器，那么request.getRemoteAddr()获取的通常是nginx的ip地址。那么有什么解决办法呢？
>The [X-Forwarded-For] [3] (XFF) HTTP header field is a [de facto standard] [2] for identifying the originating IP address of a client connecting to a web server through an HTTP proxy or load balancer. 

nginx中有一个专门的模块[ngx_http_realip_module] [1]就是用来解决这个问题的。

- X-real-ip  
应用场景:  
在客户端和应用服务之间有一台nginx，即：client->nginx->web app。那么我们只需要在配置nginx时加上  proxy_set_header  X-real-ip  $remote_addr;  那么web app通过获取请求头“X-real-ip”便可获得客户端ip。

- X-Forwarded-For  
应用场景：  
在客户端与应用服务之间有两台nginx，即：client->nginx a=>nginx b=>web app,web app希望获取client和nginx a的ip地址，则应该在nginx a和nginx b上同时配置proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 这样在web app应该会获取到：X-Forwarded-For client-ip,nginx-a-ip；而nginx b获取到的X-Forwarded-For应该为client-ip。

##X-Sendfile##
听到[X-Sendfile] [5]自然会想到sendfile，那么两者有何关联呢？都是减少copy次数。  
应用场景：
在客户端和应用服务之间有一台nginx，即：client->nginx->web app。用户希望获取web app上的一个文件，  
1.那么正常情况下文件的请求和返回顺序：client->nginx->web app->nginx->client。这样做有什么坏处呢？暂且不说web app有可能不善于处理静态文件，内存消耗，超时，断线都会造成很大的消耗。  
2.那我们为何不考虑把文件放在更善于处理静态文件的nginx上呢。
Good idea. 那我们在遇到请求文件时，直接nginx返回就行了，无需经过web app，即文件的请求和返回顺序为：client->nginx->client。  
3.但是如果我们需要权限判断怎么办呢？这时候x-sendfile就应该入场了。需要在nginx中添加如下配置。
{% highlight bash %}
location /protected/ {
  internal;
  root   /some/path;
}
{% endhighlight %} 
这时候处理顺序为：请求到达web app=>web app进行权限验证通过后，会在响应头中添加X-Accel-Redirect:path to file=>nginx 根据path to file 找到文件返回给客户端。

参考链接：  
[使用 Nginx 的 X-Sendfile 机制提升 PHP 文件下载性能] [4]  
[Nginx Wiki X-accel] [6]  
    

[1]: http://nginx.org/en/docs/http/ngx_http_realip_module.html
[2]: http://en.wikipedia.org/wiki/De_facto_standard
[3]: http://en.wikipedia.org/wiki/X-Forwarded-For
[4]: http://www.lovelucy.info/x-sendfile-in-nginx.html
[5]: http://wiki.nginx.org/XSendfile
[6]: http://wiki.nginx.org/X-accel
