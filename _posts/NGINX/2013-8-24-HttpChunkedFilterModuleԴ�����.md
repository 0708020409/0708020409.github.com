---
layout: post
title: "HttpChunkedFilterModule源码分析"
description: "Http chunked filter源码分析"
category: "NGINX源码分析"
tags: ["HTTP", "NGINX源码分析"]
---
{% include JB/setup %}


>分块传输编码（Chunked transfer encoding）是超文本传输协议（HTTP）中的一种数据传输机制，允许HTTP由网页服务器发送给客户端应用（ 通常是网页浏览器）的数据可以分成多个部分。分块传输编码只在HTTP协议1.1版本（HTTP/1.1）中提供。通常，HTTP应答消息中发送的数据是整个发送的，Content-Length消息头字段表示数据的长度。数据的长度很重要，因为客户端需要知道哪里是应答消息的结束，以及后续应答消息的开始。然而，使用分块传输编码，数据分解成一系列数据块，并以一个或多个块发送，这样服务器可以发送数据而不需要预先知道发送内容的总大小。通常数据块的大小是一致的，但也不总是这种情况。

数据格式：  
-   response line
-   response header (包括Transfer-Encoding=chunked)
-   response body  
```[Chunk大小][回车][Chunk数据体][回车][Chunk大小][回车][Chunk数据体][回车][0][回车]```



这里必须说一下chunked,content-length,keepalive和HTTP版本的关系：  

HTTP 1.0 若response header中不包含content-length 则keepalive必须为off。这是HTTP 1.0在响应头中没有content-length时判断相应终止的唯一办法。而HTTP 1.1 则有三种判断相应结束的方式：1.断开链接，即keepalive off;2.content-length标记长度；3.chunked方式。所以如果HTTP 1.1时，响应头中中不包含content-length，而且nginx配置chunked_transfer_encoding off,则nginx自动关闭keepalive。

ngx_http_chunked_header_filter实现功能：
主要是查看http版本,nginx处理方式：
-	HTTP 1.0由于响应头中没有content-length,而且http 1.0不支持chunked,这是唯一的处理方式就是关闭keepalive。
-   HTTP 1.1 查看配置选项chunked_transfer_encoding，若支持正常处理；若否，则关闭keepalive。

{% highlight c %}
static ngx_int_t
ngx_http_chunked_header_filter(ngx_http_request_t *r)
{
    ngx_http_core_loc_conf_t       *clcf;
    ngx_http_chunked_filter_ctx_t  *ctx;

    if (r->headers_out.status == NGX_HTTP_NOT_MODIFIED
        || r->headers_out.status == NGX_HTTP_NO_CONTENT
        || r->headers_out.status < NGX_HTTP_OK
        || r != r->main
        || (r->method & NGX_HTTP_HEAD))
    {
        return ngx_http_next_header_filter(r);
    }

    if (r->headers_out.content_length_n == -1) {
        if (r->http_version < NGX_HTTP_VERSION_11) {
            r->keepalive = 0;

        } else {
            clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

            if (clcf->chunked_transfer_encoding) {
                r->chunked = 1;

                ctx = ngx_pcalloc(r->pool,
                                  sizeof(ngx_http_chunked_filter_ctx_t));
                if (ctx == NULL) {
                    return NGX_ERROR;
                }

                ngx_http_set_ctx(r, ctx, ngx_http_chunked_filter_module);

            } else {
                r->keepalive = 0;
            }
        }
    }

    return ngx_http_next_header_filter(r);
}
{% endhighlight %}


ngx_http_chunked_body_filter实现功能：  
将in队列中待发送的buf平移到out队列，然后再队列前加上长度在后面加上CRLF;若出现last_buf，则加上CRLF "0" CRLF CRLF (30 0d 0a 0d 0a) 响应头表示结束。
{% highlight c %}
static ngx_int_t
ngx_http_chunked_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    u_char                         *chunk;
    off_t                           size;
    ngx_int_t                       rc;
    ngx_buf_t                      *b;
    ngx_chain_t                    *out, *cl, *tl, **ll;
    ngx_http_chunked_filter_ctx_t  *ctx;

    if (in == NULL || !r->chunked || r->header_only) {
        return ngx_http_next_body_filter(r, in);
    }

    ctx = ngx_http_get_module_ctx(r, ngx_http_chunked_filter_module);

    out = NULL;
    ll = &out;

    size = 0;
    cl = in;
    //for循环作用：1.计算此次传输数据长度；2.把in赋值给out,这里并不是copy buffer,仅仅是指针的操作。
    for ( ;; ) {
        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "http chunk: %d", ngx_buf_size(cl->buf));

        size += ngx_buf_size(cl->buf);

        if (cl->buf->flush
            || cl->buf->sync
            || ngx_buf_in_memory(cl->buf)
            || cl->buf->in_file)
        {
            tl = ngx_alloc_chain_link(r->pool);
            if (tl == NULL) {
                return NGX_ERROR;
            }

            tl->buf = cl->buf;
            *ll = tl;
            ll = &tl->next;
        }

        if (cl->next == NULL) {
            break;
        }

        cl = cl->next;
    }

    if (size) {
        tl = ngx_chain_get_free_buf(r->pool, &ctx->free);
        if (tl == NULL) {
            return NGX_ERROR;
        }

        b = tl->buf;
        chunk = b->start;

        if (chunk == NULL) {
            /* the "0000000000000000" is 64-bit hexadecimal string */

            chunk = ngx_palloc(r->pool, sizeof("0000000000000000" CRLF) - 1);
            if (chunk == NULL) {
                return NGX_ERROR;
            }

            b->start = chunk;
            b->end = chunk + sizeof("0000000000000000" CRLF) - 1;
        }

        b->tag = (ngx_buf_tag_t) &ngx_http_chunked_filter_module;
        b->memory = 0;
        b->temporary = 1;
        b->pos = chunk;
        b->last = ngx_sprintf(chunk, "%xO" CRLF, size);

        tl->next = out;
        out = tl;
    }
    //last_buf表示请求的最后一个buf,代表整个响应结束。
    if (cl->buf->last_buf) {
        tl = ngx_chain_get_free_buf(r->pool, &ctx->free);
        if (tl == NULL) {
            return NGX_ERROR;
        }

        b = tl->buf;

        b->tag = (ngx_buf_tag_t) &ngx_http_chunked_filter_module;
        b->temporary = 0;
        b->memory = 1;
        b->last_buf = 1;
        b->pos = (u_char *) CRLF "0" CRLF CRLF;
        b->last = b->pos + 7;

        cl->buf->last_buf = 0;

        *ll = tl;

        if (size == 0) {
            b->pos += 2;
        }
    //代表chunked 包的结尾，CRLF。
    } else if (size > 0) {
        tl = ngx_chain_get_free_buf(r->pool, &ctx->free);
        if (tl == NULL) {
            return NGX_ERROR;
        }

        b = tl->buf;

        b->tag = (ngx_buf_tag_t) &ngx_http_chunked_filter_module;
        b->temporary = 0;
        b->memory = 1;
        b->pos = (u_char *) CRLF;
        b->last = b->pos + 2;

        *ll = tl;

    } else {
        *ll = NULL;
    }

    rc = ngx_http_next_body_filter(r, out);

    ngx_chain_update_chains(r->pool, &ctx->free, &ctx->busy, &out,
                            (ngx_buf_tag_t) &ngx_http_chunked_filter_module);

    return rc;
}
{% endhighlight %}
Talk is cheap.Let's see it.  
测试使用的是nginx+lua模块，可以直接使用[openresty] [2]。  
配置文件：
{% highlight bash%}
location /var {
    default_type text/html;
    content_by_lua '
        ngx.say("ngx.var.host:"..ngx.var.host);
        ngx.say("ngx.var.request_method:"..ngx.var.request_method);
        ngx.say("ngx.var.args:"..ngx.var.args);
        ngx.say("ngx.var.remote_addr:"..ngx.var.remote_addr);
        ngx.say("ngx.var.remote_port:"..ngx.var.remote_port);
        -- ngx.say("ngx.var.remote_user:"..ngx.var.remote_user);
        ngx.say("ngx.var.server_name:"..ngx.var.server_name);
        ngx.say("ngx.var.server_port:"..ngx.var.server_port);
        ngx.say("ngx.var.server_addr:"..ngx.var.server_addr);
        ngx.say("ngx.var.uri:"..ngx.var.uri);
        ngx.say("ngx.var.request_uri:"..ngx.var.request_uri);
        ngx.say("ngx.var.query_string:"..ngx.var.query_string);
        ngx.say("ngx.var.scheme:"..ngx.var.scheme);
        ngx.say("ngx.var.server_protocol:"..ngx.var.server_protocol);
    ';
}
{% endhighlight %}

Curl请求结果  
{% highlight bash%}
[root@localhost 0708020409.github.com]# curl -v http://localhost/var?tt=1
* About to connect() to localhost port 80 (#0)
*   Trying ::1... 拒绝连接
*   Trying 127.0.0.1... connected
* Connected to localhost (127.0.0.1) port 80 (#0)
> GET /var?tt=1 HTTP/1.1
> User-Agent: curl/7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.14.0.0 zlib/1.2.3 libidn/1.18 libssh2/1.4.2
> Host: localhost
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: ngx_openresty/1.4.2.8
< Date: Mon, 24 Mar 2014 10:27:45 GMT
< Content-Type: text/html
< Transfer-Encoding: chunked
< Connection: keep-alive
< 
ngx.var.host:localhost
ngx.var.request_method:GET
ngx.var.args:tt=1
ngx.var.remote_addr:127.0.0.1
ngx.var.remote_port:44166
ngx.var.server_name:_
ngx.var.server_port:80
ngx.var.server_addr:127.0.0.1
ngx.var.uri:/var
ngx.var.request_uri:/var?tt=1
ngx.var.query_string:tt=1
ngx.var.scheme:http
ngx.var.server_protocol:HTTP/1.1
* Connection #0 to host localhost left intact
* Closing connection #0
{% endhighlight %}

wireshark过滤nginx返回包：
```ip.src== 192.168.65.134 && http```

![enter image description here][1]   
如图中标记部分：  
```31 62  0d 0a 6e 67 78 2e 76 61 72 2e 72 65 71 75 65 73  74 5f 6d 65 74 68 6f 64 3a 47 45 54 0a 0d 0a```

31 62 代表此chunk的长度：0x1b 即27个字节；然后跟着分隔符CRLF;再次就是数据27个字节;最后又是分隔符CRLF。

[1]: http://192.168.65.134:4000/img/HttpChunkedFilterModule.png
[2]: http://openresty.org/