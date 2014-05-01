---
layout: post
title: "websocket协议学习"
description: "websocket推送"
category: "websocket,html5"
tags: [websocket]
---
{% include JB/setup %}


Talk is cheap.先看实例：

{% highlight bash %}
wget http://openresty.org/download/ngx_openresty-1.5.8.1.tar.gz
tar xzvf ngx_openresty-1.5.8.1.tar.gz
cd ngx_openresty-1.5.8.1
./configure --with-luajit
{% endhighlight %}

websocket.html
{% highlight html %}
<html>
    <head>
        <script>
            var ws = null;
            function connect() {
                if (ws !== null) return log('already connected');
                ws = new WebSocket('ws://192.168.65.138/s');
                ws.onopen = function() {
                    log('connected');
                };
                ws.onerror = function(error) {
                    log('error:' + error);
                };
                ws.onmessage = function(e) {
                    log('recv: ' + e.data);
                };
                ws.onclose = function() {
                    log('disconnected');
                    ws = null;
                };
                return false;
            }
            function disconnect() {
                if (ws === null) return log('already disconnected');
                ws.close();
                return false;
            }
            function send() {
                if (ws === null) return log('please connect first');
                var text = document.getElementById('text').value;
                document.getElementById('text').value = "";
                log('send: ' + text);
                ws.send(text);
                return false;
            }
            function log(text) {
                var li = document.createElement('li');
                li.appendChild(document.createTextNode(text));
                document.getElementById('log').appendChild(li);
                return false;
            }
        </script>
    </head>
    
    <body>
        <form onsubmit="return send();">
            <button type="button" onclick="return connect();">
                Connect
            </button>
            <button type="button" onclick="return disconnect();">
                Disconnect
            </button>
            <input id="text" type="text">
            <button type="submit">
                Send
            </button>
        </form>
        <ol id="log">
        </ol>
    </body>

</html>
{% endhighlight %}

nginx.conf
{% highlight bash %}
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}



http {
    include       mime.types;
    default_type  application/octet-stream;

    lua_package_path "/root/github/nginx-study/openrestry/lualib/resty/websocket/?.lua;;";

    gzip on;
    gzip_min_length 1;
 
    server {
        listen 80  so_keepalive=2s:2s:8;

        location /s {
            lua_socket_log_errors  on;
            lua_check_client_abort on;
            content_by_lua '
                local server = require "resty.websocket.server"
                local wb, err = server:new{
                    timeout = 50000,
                    max_payload_len = 65535,
                }
                if not wb then
                    ngx.log(ngx.ERR, "failed to new websocket: ", err)
                    return ngx.exit(444)
                end
                while true do
                    local data, typ, err = wb:recv_frame()
                    if wb.fatal then
                        ngx.log(ngx.ERR, "failed to receive frame: ", err)
                        return ngx.exit(444)
                    end
                    if not data then
                        local bytes, err = wb:send_ping()
                        if not bytes then
                            ngx.log(ngx.ERR, "failed to send ping: ", err)
                            return ngx.exit(444)
                        end
                    elseif typ == "close" then break
                    elseif typ == "ping" then
                        local bytes, err = wb:send_pong()
                        if not bytes then
                            ngx.log(ngx.ERR, "failed to send pong: ", err)
                            return ngx.exit(444)
                        end
                    elseif typ == "pong" then
                        ngx.log(ngx.INFO, "client ponged")
                    elseif typ == "text" then
                        local bytes, err = wb:send_text(data)
                        if not bytes then
                            ngx.log(ngx.ERR, "failed to send text: ", err)
                            return ngx.exit(444)
                        end
                    end
                end
                wb:send_close()
            ';
        }
    }
}
{% endhighlight %}

既然是websocket协议，那么就需要从http协议转化为websocket协议
{% highlight bash %}
GET http://192.168.65.138/s HTTP/1.1
Origin: http://192.168.65.138
Sec-WebSocket-Key: D0QLjvO4fnHcXaE0/gqn6A==
Connection: Upgrade
Upgrade: Websocket
Sec-WebSocket-Version: 13
User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64; Trident/7.0; rv:11.0) like Gecko
Host: 192.168.65.138
DNT: 1
Cache-Control: no-cache
{% endhighlight %}

{% highlight bash %}
HTTP/1.1 101 Switching Protocols
Server: openresty/1.5.8.1
Date: Thu, 10 Apr 2014 06:52:07 GMT
Connection: upgrade
Upgrade: websocket
Sec-WebSocket-Accept: OedKikXLqz4gzGARLjmhZzRcNjk=
{% endhighlight %}

具体实现可以看春哥的[lua-resty-websocket] [2]，实际上就是lua通过ngx.req.socket接管http处理的socket，然后在此socket上实现websocket协议。剩余的都是协议[RFC6455] [1]的实现。

嘎嘎嘎，这里得瑟下，我给春哥提了个bug。虽然是个简单的[bug] [3]，但是嘎嘎嘎嘎，现在的我迫切需要得到别人的肯定。


[1]: http://tools.ietf.org/html/rfc6455
[2]: https://github.com/agentzh/lua-resty-websocket
[3]: https://github.com/agentzh/lua-resty-websocket/commit/ac37f98
