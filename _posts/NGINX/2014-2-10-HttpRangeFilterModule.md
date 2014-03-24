---
layout: post
title: "HttpRangeFilterModule源码分析"
description: "HttpRangeFilterModule源码分析"
category: "NGINX源码分析"
tags: ["HTTP", "NGINX源码分析"]
---
{% include JB/setup %}
了解nginx filter之前，先了解下curl命令吧。
{% highlight bash %}
curl -v http://localhost/nginx_status  
curl -I http://localhost/nginx_status  
curl -X HEAD -v http://localhost/nginx_status  
curl -v -r "0-1,500-" http://localhost/  
{% endhighlight %}

啥都不说，先感受下range的作用：
{% highlight bash %}
[root@localhost sbin]# curl -v -r "0-1,500-" http://localhost/
* About to connect() to localhost port 80 (#0)
*   Trying ::1... Connection refused
*   Trying 127.0.0.1... connected
* Connected to localhost (127.0.0.1) port 80 (#0)
> GET / HTTP/1.1
> Range: bytes=0-1,500-
> User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.14.0.0 zlib/1.2.3 libidn/1.18 libssh2/1.4.2
> Host: localhost
> Accept: */*
> 
< HTTP/1.1 206 Partial Content
< Server: nginx/1.4.2
< Date: Mon, 24 Mar 2014 13:47:04 GMT
< Content-Type: multipart/byteranges; boundary=00000000000000000007
< Content-Length: 312
< Last-Modified: Fri, 29 Nov 2013 08:17:43 GMT
< Connection: keep-alive
< ETag: "52984da7-264"
< 

--00000000000000000007
Content-Type: text/html
Content-Range: bytes 0-1/612

<!
--00000000000000000007
Content-Type: text/html
Content-Range: bytes 500-611/612

e at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

--00000000000000000007--
* Connection #0 to host localhost left intact
* Closing connection #0
{% endhighlight %}

粗略的看了下代码，发现没什么好解析的，重点是理解协议格式。
有一点就是：
在if_range验证失败时，会返回Accept_Ranges：bytes。为什么呢。主要是告诉客户端，我nginx不是不支持if_range，而是验证失败。