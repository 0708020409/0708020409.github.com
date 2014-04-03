---
layout: post
title: "nginx进程间通讯和信号处理"
description: "socketpair IPC"
category: "网络编程"
tags: [网络编程]
---
{% include JB/setup %}

有两种方式对nginx发送信号:
1. ```./nginx -s reload``` 
2. ```kill -USR1 $NGINX_PID```  

{% highlight c %}
if (ngx_signal) {
    return ngx_signal_process(cycle, ngx_signal);
}

ngx_os_status(cycle->log);

ngx_cycle = cycle;

ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);

if (ccf->master && ngx_process == NGX_PROCESS_SINGLE) {
    ngx_process = NGX_PROCESS_MASTER;
}

#if !(NGX_WIN32)

if (ngx_init_signals(cycle->log) != NGX_OK) {
    return 1;
}
{% endhighlight %}

其中以```./nginx -s reload```其实呢，就是nginx自己调用```kill -USR1 $NGINX_PID```。  
具体分析：  
nginx解析命令行，发现有-s，将ngx_signal置1。而后如上调用ngx_signal_process。实际上就是去查找nginx.pid，然后对该进程调用kill。这里需要注意的，如果nginx在启动时，启用了-s选项，则进程处理完之后直接退出。  


可以从上面代码中看见nginx其实是先处理命令行中的信号，然后初始化信号处理函数。那如果，之前并没有启动nginx，调用```./nginx -s reload```的后果是什么呢？  
{% highlight bash %}
[root@localhost sbin]# ./nginx -s reload
nginx: [error] open() "/root/github/nginx-study/nginx/logs/nginx.pid" failed (2: No such file or directory)
{% endhighlight %}

ngx_init_signals实际上就是把要处理的信号的处理函数都设置为ngx_signal_handler。

{% highlight c %}
switch (ngx_process) {

case NGX_PROCESS_MASTER:
case NGX_PROCESS_SINGLE:
    switch (signo) {

    case ngx_signal_value(NGX_SHUTDOWN_SIGNAL):
        ngx_quit = 1;
        action = ", shutting down";
        break;

    case ngx_signal_value(NGX_TERMINATE_SIGNAL):
    case SIGINT:
        ngx_terminate = 1;
        action = ", exiting";
        break;
{% endhighlight %}

如上代码，比如使用```kill -QUIT $NGINX_PID```，信号处理函数会置ngx_quit为1。

然后呢？这里要看master process的主循环-ngx_master_process_cycle。         
这个函数主要完成两个任务:
-   启动worker process,cache manager process； 
-   for(;;)检测如ngx_quit的值，并执行相应动作。

比如：
{% highlight c %}
if (ngx_quit) {
    ngx_signal_worker_processes(cycle,
                    ngx_signal_value(NGX_SHUTDOWN_SIGNAL));

    ls = cycle->listening.elts;
    for (n = 0; n < cycle->listening.nelts; n++) {
        if (ngx_close_socket(ls[n].fd) == -1) {
            ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_socket_errno,
                            ngx_close_socket_n " %V failed",
                                &ls[n].addr_text);
         }
    }
    cycle->listening.nelts = 0;

    continue;
}
{% endhighlight %}

那么来看ngx_signal_worker_processes的实现
{% highlight c %}
if (ngx_processes[i].detached || ngx_processes[i].pid == -1) {
    continue;
}

if (ngx_processes[i].just_spawn) {
    ngx_processes[i].just_spawn = 0;
    continue;
}

if (ngx_processes[i].exiting
    && signo == ngx_signal_value(NGX_SHUTDOWN_SIGNAL))
{
    continue;
}

if (ch.command) {
    if (ngx_write_channel(ngx_processes[i].channel[0],
                        &ch, sizeof(ngx_channel_t), cycle->log) == NGX_OK)
    {
        if (signo != ngx_signal_value(NGX_REOPEN_SIGNAL)) {
            ngx_processes[i].exiting = 1;
        }

        continue;
    }
}

ngx_log_debug2(NGX_LOG_DEBUG_CORE, cycle->log, 0,
                "kill (%P, %d)" , ngx_processes[i].pid, signo);

if (kill(ngx_processes[i].pid, signo) == -1) {
    err = ngx_errno;
    ngx_log_error(NGX_LOG_ALERT, cycle->log, err,
                    "kill(%P, %d) failed", ngx_processes[i].pid, signo);

    if (err == NGX_ESRCH) {
        ngx_processes[i].exited = 1;
        ngx_processes[i].exiting = 0;
        ngx_reap = 1;
    }

    continue;
}

if (signo != ngx_signal_value(NGX_REOPEN_SIGNAL)) {
    ngx_processes[i].exiting = 1;
}
{% endhighlight %}
这里是通过ngx_write_channel进行通讯。曾经有人问我，nginx worker进程是通过epoll来侦听读写事件的，那是通过什么侦听channel的读事件呢？
也是epoll!可是这里
{% highlight c %}
if (ioctl(ngx_processes[s].channel[0], FIOASYNC, &on) == -1) {
    ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                    "ioctl(FIOASYNC) failed while spawning \"%s\"", name);
    ngx_close_channel(ngx_processes[s].channel, cycle->log);
    return NGX_INVALID_PID;
}

if (fcntl(ngx_processes[s].channel[0], F_SETOWN, ngx_pid) == -1) {
    ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                    "fcntl(F_SETOWN) failed while spawning \"%s\"", name);
    ngx_close_channel(ngx_processes[s].channel, cycle->log);
    return NGX_INVALID_PID;
}

if (fcntl(ngx_processes[s].channel[0], F_SETFD, FD_CLOEXEC) == -1) {
    ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                    "fcntl(FD_CLOEXEC) failed while spawning \"%s\"",
                    name);
    ngx_close_channel(ngx_processes[s].channel, cycle->log);
    return NGX_INVALID_PID;
}

if (fcntl(ngx_processes[s].channel[1], F_SETFD, FD_CLOEXEC) == -1) {
    ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                    "fcntl(FD_CLOEXEC) failed while spawning \"%s\"",
                    name);
    ngx_close_channel(ngx_processes[s].channel, cycle->log);
    return NGX_INVALID_PID;
}
{% endhighlight %}
但是这个地方是干啥的呢？
>FIOASYNC Enables a simple form of asynchronous I/O notification. This command causes the kernel to send SIGIO signal to a process or a process group when I/O is possible. Only sockets, ttys, and pseudo-ttys implement this functionality.

>FIONBIO Enables nonblocking I/O. The effect is similar to setting the O_NONBLOCK flag with the fcntl subroutine. The third parameter to the ioctl subroutine for this command is a pointer to an integer that indicates whether nonblocking I/O is being enabled or disabled. A value of 0 disables non-blocking I/O.

按照我的理解，应该是nginx其实可以读取子进程发给父进程的东西，只是暂时没有使用到。所以在ngx_signal_handler中
{% highlight c %}
case SIGIO:
    ngx_sigio = 1;
    break;
{% endhighlight %}
在收到SIGIO时，会设置ngx_sigio为1。而ngx_sigio再没在其他地方出现过。注意这个是主进程，而子进程是忽略该信号的。  
所以其实nginx使用了两种方式进行检测是否有channel可读:
-   1.epoll;
-   2.sigio。  

在ngx_write_channel失败时呢，nginx会使用kill发送信号给worker进程。那么子进程如何处理呢？  
ngx_write_channel成功后，nginx子进程epoll_wait侦听到读事件，ngx_channel_handler进行处理。以quit为例，这时会设置ngx_quit=1。而在ngx_worker_process_cycle的for循环中，
{% highlight c %}
for ( ;; ) {

    if (ngx_exiting) {

        c = cycle->connections;

        for (i = 0; i < cycle->connection_n; i++) {

            /* THREAD: lock */

            if (c[i].fd != -1 && c[i].idle) {
                c[i].close = 1;
                c[i].read->handler(c[i].read);
            }
        }

        if (ngx_event_timer_rbtree.root == ngx_event_timer_rbtree.sentinel)
        {
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");

            ngx_worker_process_exit(cycle);
        }
    }

    ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, "worker cycle");

    ngx_process_events_and_timers(cycle);

    if (ngx_terminate) {
        ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");

        ngx_worker_process_exit(cycle);
    }

    if (ngx_quit) {
        ngx_quit = 0;
        ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0,
                        "gracefully shutting down");
        ngx_setproctitle("worker process is shutting down");

        if (!ngx_exiting) {
            ngx_close_listening_sockets(cycle);
            ngx_exiting = 1;
        }
    }

    if (ngx_reopen) {
        ngx_reopen = 0;
        ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reopening logs");
        ngx_reopen_files(cycle, -1);
    }
}
{% endhighlight %}

这里可以看到在ngx_quit=1时，nginx会首先调用ngx_close_listening_socket,此函数实际上做的就是把侦听事件从epoll中取出，即不再接受新的请求。而后，置ngx_exiting=1,循环处理剩余事件，直到把红黑树中的请求处理完。
ok.这就是nginx对于quit的大概处理流程。


接下来分析一下：
-   热升级；
-   重新加载配置文件；

热升级是怎么做的呢？  
1.编译新Nginx源码，安装路径需与旧版一致；  
2.向主进程发送```USR2```信号，Nginx会启动一个新版本的master进程和工作进程，和旧版一起处理请求；
{% highlight bash %}
[root@localhost sbin]# ps -ef |grep nginx|grep -v grep
root      3262     1  0 18:49 ?        00:00:00 nginx: master process ./nginx
nobody    3263  3262  0 18:49 ?        00:00:00 nginx: worker process
[root@localhost sbin]# kill -USR2 3262
[root@localhost sbin]# ps -ef |grep nginx|grep -v grep
root      3262     1  0 18:49 ?        00:00:00 nginx: master process ./nginx
nobody    3263  3262  0 18:49 ?        00:00:00 nginx: worker process
root      3270  3262  0 18:49 ?        00:00:00 nginx: master process ./nginx
nobody    3275  3270  0 18:50 ?        00:00:00 nginx: worker process
{% endhighlight %}
3.向原Nginx主进程发送```WINCH```信号，它会逐步关闭旗下的工作进程（主进程不退出），这时所有请求都会由新版Nginx处理；  
{% highlight bash %}
[root@localhost sbin]# kill -WINCH 3262
[root@localhost sbin]# ps -ef |grep nginx|grep -v grep
root      3262     1  0 18:49 ?        00:00:00 nginx: master process ./nginx
root      3270  3262  0 18:49 ?        00:00:00 nginx: master process ./nginx
nobody    3275  3270  0 18:50 ?        00:00:00 nginx: worker process
{% endhighlight %}
4.如果这时需要回退，可向原Nginx主进程发送```HUP```信号，它会重新启动工作进程， 仍使用旧版配置文件 。尔后可以将新版Nginx进程杀死（使用QUIT、TERM、或者KILL）；  
5.如果不需要回滚，可以将原Nginx主进程杀死，至此完成热升级。

有没有什么疑问呢？发送```USR2```，Nginx会启动一个新版本的master进程和工作进程，为什么没有出现```Address already in use```呢？这不科学。对吧。  
好吧，我承认这是曾经一个面试官问我的，而且我没有回答出来。  
```SO_REUSEADDR```会不会是它呢？
>This socket option tells the kernel that even if this port is busy (in the TIME_WAIT state), go ahead and reuse it anyway.  If it is busy,but with another state, you will still get an address already in use error.  
显然不是。  
好吧，还是看代码吧.  
execv之后如何共享fd?通过环境变量传递fd。  
```USR2=>ngx_signal_handler=>ngx_change_binary=1=>ngx_exec_new_binary=>ngx_execute=>ngx_spawn_process=>ngx_execute_proc=>ngx_spawn_process=>execve```
可以看到最后调用了execve。被打开的文件描述符是否被将会被继承呢？  
FD_CLOSEXEC close on exec, not on-fork  
若文件描述符fd没有设置FD_CLOSEXEC,则execv之后还可以继续使用该fd。那么如果我们将侦听的fd传给新的进程就可以了，但是如何传递呢？nginx是用的环境变量的方式。  

而发送WINCH信号是如何处理呢？ngx_noaccept=1,nginx给worker进程发送quit信号。ngx_signal_worker_processes(cycle,ngx_signal_value(NGX_SHUTDOWN_SIGNAL));这里其实WINCH信号会将worker进程杀死！剩下master进程。


waitpid SIGCHILD处理

>On Unix and Unix-like computer operating systems, a zombie process or defunct process is a process that has completed execution but still has an entry in the process table. 

>To remove zombies from a system, the SIGCHLD signal can be sent to the parent manually, using the kill command. If the parent process still refuses to reap the zombie, the next step would
>be to remove the parent process. When a process loses its parent, init becomes its new parent. init periodically executes the wait system call to reap any zombies with init as parent.

>If the parent explicitly ignores SIGCHLD by setting its handler to SIG_IGN (rather than simply ignoring the signal by default) or has the SA_NOCLDWAIT flag set, all child exit status
>information will be discarded and no zombie processes will be left.


对与进程关闭后的处理：
WTERMSIG