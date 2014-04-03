---
layout: post
title: "gzip使用指南"
description: "gzip, deflate，压缩"
category: "网络编程"
tags: [网络编程]
---
{% include JB/setup %}

>chunked + gzip 可以实现 chunked 本来的功能么？
>1. chunked 分块传输到浏览器，可以先把 css 传输给浏览器然后让浏览器进行 css 的下载，跟内容的加载并行，提高显示速度
>2. 目前搜索到的资料显示，如果chunked和 gzip一起用，那么浏览器也是得到所有的 chunk 之后，再进行解压，才能进行解析
>问题：
>如果 chunked 和 gzip 一起使用的时候，还能够实现 1 那样的功能么？

如上，在逛[知乎] [1]的时候,遇到以上问题。对于提问者的疑问我真是不能再赞同了，我也有同样的感觉。但是事实是怎样的呢？    
抱着试一试的态度，在openresty的邮件列表中问了下[春哥] [2]，毕竟春哥。一语惊醒梦中人，我还是too young.too simple.Gzip是流式的呀。当然同时春哥也给出了一个测试用例。
于是乎，不管后面怎么说。先试了再说。
{% highlight bash %}
http {
    gzip on;
    gzip_min_length 1;
 
    server {
        listen 8080;
 
        location = /t {
        default_type text/html;
        content_by_lua '
        local rep = string.rep
        local say = ngx.say
        local sleep = ngx.sleep
        local flush = ngx.flush
        for i = 1, 10 do
            say(rep("hello world", 1))
            say(rep("=", 2))
            flush(true)
            sleep(1)
        end
        ';
    }
}
{% endhighlight %}
怎么回事？显然这不科学。
{% highlight bash %}
[root@localhost sbin]# curl --compressed http://localhost/t
hello world
==
curl: (18) transfer closed with outstanding read data remaining
{% endhighlight %}
难道是bug?有可能是版本问题！于是找了个最新版本，果然：
{% highlight bash %}
[root@localhost sbin]# curl --compressed -v http://localhost/t
* About to connect() to localhost port 80 (#0)
*   Trying ::1... Connection refused
*   Trying 127.0.0.1... connected
* Connected to localhost (127.0.0.1) port 80 (#0)
> GET /t HTTP/1.1
> User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.14.0.0 zlib/1.2.3 libidn/1.18 libssh2/1.4.2
> Host: localhost
> Accept: */*
> Accept-Encoding: deflate, gzip
> 
< HTTP/1.1 200 OK
< Server: openresty/1.5.8.1
< Date: Tue, 25 Mar 2014 14:14:07 GMT
< Content-Type: text/html
< Transfer-Encoding: chunked
< Connection: keep-alive
< Content-Encoding: gzip
< 
hello world
==
hello world
==
hello world
==
hello world
==
hello world
==
* Connection #0 to host localhost left intact
* Closing connection #0
{% endhighlight %}
额。这个结果就是hello world每秒出来1个。

[1]: http://www.zhihu.com/question/21251382/answer/19426202
[2]: http://weibo.com/agentzh