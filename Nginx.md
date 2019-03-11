# 一. 基本介绍
负载均衡涉及到以下的基础知识。

## 1. 负载均衡算法

a. Round Robin(**轮询**): 对所有的backend轮询发送请求，每个请求**按时间顺序**逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。默认的分配方式；

b.**指定权重**：指定**轮询几率**，weight和访问比率成正比，用于后端服务器性能不均的情况。 


c. **fair（第三方）** 按后端服务器的响应时间来分配请求，响应时间短的优先分配.

​	**Least Connections(least_conn)**: 跟踪和backend当前的活跃连接数目，最少的连接数目说明这个backend		负载最轻，将请求分配给他，这种方式会考虑到配置中给每个upstream分配的weight权重信息；

​	**Least Time(least_time):** 请求会分配给响应最快和活跃连接数最少的backend；


d. **IP Hash(ip_hash):** 对请求来源IP地址计算hash值，IPv4会考虑前3个octet，IPv6会考虑所有的地址位，然后根据得到的hash值通过某种映射分配到backend；

e. **Generic Hash(hash):** 以用户自定义资源(比如URL)的方式计算hash值完成分配，其可选consistent关键字支持一致性hash特性，每个url定向到同一个后端服务器，后端服务器为缓存时比较有效；



## 2. **会话一致性**

用户(浏览器)在和服务端交互的时候，通常会在本地保存一些信息，而整个过程叫做一个会话(Session)并用唯一的Session ID进行标识。因为HTTP协议是无状态的，所以任何需要逻辑上下文的情形都必须使用会话机制，此外HTTP客户端也会额外缓存一些数据在本地，这样就可以减少请求提高性能了。如果负载均衡可能将这个会话的请求分配到不同的后台服务端上，这肯定是不合适的，必须通过多个backend共享这些数据，效率肯定会很低下，**最简单**的情况是保证会话一致性——**相同的会话每次请求都会被分配到同一个backend上**去。

目前支持三种模式的会话一致性：

### 2.1.  **Cookie Insertion**

在backend第一次response之后，会在其头部添加一个session cookie，之后客户端接下来的请求都会带有这个cookie值，Nginx可以根据这个cookie判断需要转发给哪个backend了。

```
sticky cookie srv_id expires=1h domain=.example.com path=/;
```

上面的srv_id代表了cookie的名字，而后面的参数expires、domain、path都是可选的。



### 2.2.  **Sticky Routes**

也是在backend第一次response之后，会产生一个route信息，route信息通常会从cookie/URI信息中提取。

```
sticky route $route_cookie $route_uri;
```

这样Nginx会按照顺序搜索$route_cookie、$route_uri参数并选择第一个非空的参数用作route，而如果所有的参数都是空的，就使用上面默认的负载均衡算法决定请求分发给哪个backend。



### 2.3. **Learn**

较为的复杂也较为的智能，Nginx会自动监测request和response中的session信息，而且通常需要回话一致性的请求、应答中都会带有session信息，这和第一种方式相比是不用增加cookie，而是动态学习已有的session。

这种方式需要使用到zone结构，在Nginx中zone都是共享内存，可以在多个worker process中共享数据用的。(不过其他的会话一致性怎么没用到共享内存区域呢？)

```
sticky learn 
   create=$upstream_cookie_examplecookie
   lookup=$cookie_examplecookie
   zone=client_sessions:1m
   timeout=1h;
```



## 3. **后台服务端的动态配置**

出问题的backend要能被及时探测并剔除出分配群，而当业务增长的时候可以灵活的添加backend数目。

有需要关闭某些backend以便维护或者升级时请求不会发送到这个backend上面，而之前分配到这个backend的会话的后续请求还会继续发送给他，直到这个会话最终完成。

让某个backend进入draining的状态，既可以直接修改配置文件，然后通过向master process发送信号重新加载配置，也可以采用Nginx的on-the-fly配置方式。



## 4.  **基于DNS的负载均衡**

通常现代的网络服务者一个域名会关连到多个主机，在进行DNS查询的时候，默认情况下DNS服务器会以**round-robin**形式以不同的顺序返回IP地址列表，因此天然将客户请求分配到不同的主机上去。

不过这种方式含有固有的缺陷：DNS不会检查主机和IP地址的可访问性，所以分配给客户端的IP不确保是可用的(404)；DNS的解析结果会在客户端、多个中间DNS服务器不断的缓存，所以backend的分配不会那么的理想。



# 二. 配置文件

## 1. 负载均衡配置



**Round Robin(轮询):**

```conf
upstream backserver { 
    server 192.168.0.14; 
    server 192.168.0.15; 
} 
```

**指定权重**
```config
upstream backserver { 
    server 192.168.0.14 weight=10; 
    server 192.168.0.15 weight=10; 
} 
```

**fair（第三方）**

```config
upstream backserver {
    server server1;
    server server2;
    fair;
} 
```

**Generic Hash**
```config
upstream backserver {
    server squid1:3128;
    server squid2:3128;
    hash $request_uri;
    hash_method crc32;
} 
```

**Ip Hash**

```config
proxy_pass http://backserver/;
upstream backserver{
    ip_hash;
    server 127.0.0.1:9090 down; (down 表示单前的server暂时不参与负载)
    server 127.0.0.1:8080 weight=2; (weight 默认为1.weight越大，负载的权重就越大)
    server 127.0.0.1:6060;
    server 127.0.0.1:7070 backup; (其它所有的非backup机器down或者忙的时候，请求backup机器)
}

max_fails ：允许请求失败的次数默认为1.当超过最大次数时，返回proxy_next_upstream 模块定义的错误
fail_timeout:max_fails次失败后，暂停的时间
```



## 2. 代理服务器配置



```xml
#操作系统启动多少个工作进程运行Nginx
worker_processes  1; 

events {
    worker_connections  1024;
}


http {

	upstream shopservice {
		server localhost:8081;
		server 172.18.11.215:8081 weight=2;
		server 172.18.11.225:8081 weight=2;
	}

    include       mime.types;						#文件扩展名与文件类型映射表
	default_type application/octet-stream;			#默认文件类型 无特定类型的 二进制流文件
    sendfile        on;
    keepalive_timeout  65;
	
	server {
	   listen 80;
	   server_name _;
	   
	   return 404;
	}

	#开启nginx状态监控
    #server {
    #    listen  *:80 default_server;
    #    server_name _;
    #    location /ngx_status  
    #    {
    #        stub_status on;
    #        access_log off;
    #        #allow 127.0.0.1;
    #        #deny all;
    #    }
    #}
	
    server {
        listen       	80;
        server_name  	localhost;		

        location / {
            proxy_pass   http://shopservice;
            index  index.html index.htm;
        }  		
    }
}

```



--------------------------

**proxy_pass**配置说明：

假如要访问http://192.168.1.4/proxy/test.html ，有四种情况。

1. 第一种

   ```xml
   location  /proxy/ {
   	proxy_pass http://127.0.0.1:81/;
   }
   ```

   结论：会被代理到http://127.0.0.1:81/test.html 这个url

2. 第二种(相对于第一种，最后少一个 /)

   ```xml
   location  /proxy/ {
   	proxy_pass http://127.0.0.1:81;
   }
   ```

   结论：会被代理到http://127.0.0.1:81/proxy/test.html 这个url


3. 第三种

   ```xml
   location  /proxy/ {
   	proxy_pass http://127.0.0.1:81/ftlynx/;
   }
   ```

   结论：会被代理到http://127.0.0.1:81/ftlynx/test.html 这个url。



3. 第四种(相对于第三种，最后少一个 / )：

   ```xml
   location  /proxy/ {
   	proxy_pass http://127.0.0.1:81/ftlynx;
   }
   ```

   结论：会被代理到http://127.0.0.1:81/ftlynxtest.html 这个url



**proxy_pass是否带 / 问题**

> ```
> http://abc.com:8080 写法和 http://abc.com:8080/ 写法的区别如下:
> 
>     不带/
>     location /NginxTest/ {
>     	proxy_pass  http://abc.com:8080;
>     }
> 
>     带/
>     location /NginxTest/ {
>         proxy_pass  http://abc.com:8080/;
>     }
>     上面两种配置，区别只在于proxy_pass转发的路径后是否带 “/”。
>     针对情况1:
>             (带参数)如果访问url =http://localhost:90/NginxTest/servlet/MyServlet?name=123333333，则被nginx代理后，请求路径便会访问http://abc.com:8080/NginxTest/servlet/MyServlet?name=123333333。
>            （不带参数）如果访问url =http://localhost:90/NginxTest/servlet/MyServlet，则被nginx代理后，请求路径便会访问http://abc.com:8080/NginxTest/servlet/MyServlet。
> 
>     针对情况2：
>               如果访问url = http://server/NginxTest/test.jsp，则被nginx代理后，请求路径会变为 http://proxy_pass/test.jsp，直接访问server的根资源。
>               访问http://localhost:90/NginxTest/NginxTest/NginxTest/servlet/MyServlet，被nginx代理后，请求路径才会访问http://abc.com:8080/NginxTest/servlet/MyServlet。
>     **注意：上面两种访问路径的差别。
>     修改配置后重启nginx代理就成功了。**
> ```



-----------

**location**配置说明：

>location [=|~|~*|^~] patt { } 
>
>=:严格匹配。如果这个查询匹配，那么将停止搜索并立即处理此请求。
>
>~:为区分大小写匹配(可用正则表达式)。
>
>~*:为不区分大小写匹配(可用正则表达式)。
>
>^~:如果把这个前缀用于一个常规字符串,那么告诉nginx 如果路径匹配那么不测试正则表达式。



**location**匹配顺序：

1. 如果精准命中，立刻返回结果。
2. 判断普通命中，如果只有一个命中，记录结果并继续匹配。如果有多个命中，记录最长命中结果并继续匹配。
3. 判断正则匹配，按配置文件中从上到下的顺序进行匹配，一旦匹配成功则返回。
4. 如果正则匹配失败则使用普通命中的结果。



​	其中用普通字符串配置的location顺序是无关紧要的，但是需要注意的是正则表达式按照配置文件里的顺序测试。找到第一个匹配的正则表达式将停止搜索。

​	一般情况下，匹配成功了普通字符串location后还会进行正则表达式location匹配。有两种方法改变这种行为，其一就是使用“=”前缀，这时执行的是严格匹配，并且匹配成功后立即停止其他匹配，同时处理这个请求；另外一种就是使用“^~”前缀，如果把这个前缀用于一个常规字符串那么告诉nginx 如果路径匹配那么不测试正则表达式。

![location匹配顺序](截图\Nginx\location匹配顺序.jpg)

//todo 图需要改

-------------


## 3. 部分配置说明

**1. worker_processes**：

操作系统启动多少个工作进程运行Nginx。注意是工作进程，不是有多少个nginx工程。在Nginx运行的时候，会启动两种进程，一种是主进程master process；一种是工作进程worker process。例如我在配置文件中将worker_processes设置为4，启动Nginx后，使用进程查看命令观察名字叫做nginx的进程信息，会看到如下结果

```log
[root@localhost nginx]# ps -elf | grep nginx 
4 S root 2203 2031 0 80 0 - 46881 wait 22:18 pts/0 00:00:00 su nginx 
4 S nginx 2204 2203 0 80 0 - 28877 wait 22:18 pts/0 00:00:00 bash 
5 S root 2252 1 0 80 0 - 11390 sigsus 22:20 ? 00:00:00 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf 
5 S nobody 2291 2252 0 80 0 - 11498 ep_pol 22:23 ? 00:00:00 nginx: worker process
5 S nobody 2292 2252 0 80 0 - 11498 ep_pol 22:23 ? 00:00:00 nginx: worker process 
5 S nobody 2293 2252 0 80 0 - 11498 ep_pol 22:23 ? 00:00:00 nginx: worker process 
5 S nobody 2294 2252 0 80 0 - 11498 ep_pol 22:23 ? 00:00:00 nginx: worker process
0 R root 2312 2299 0 80 0 - 28166 - 22:24 pts/0 00:00:00 grep --color=auto nginx
```

图中可以看到1个nginx主进程，master process；还有四个工作进程，worker process。主进程负责监控端口，协调工作进程的工作状态，分配工作任务，工作进程负责进行任务处理。一般这个参数要和操作系统的CPU内核数成倍数。



**2. worker_connections**：

这个属性是指单个工作进程可以允许同时建立外部连接的数量。无论这个连接是外部主动建立的，还是内部建立的。这里需要注意的是，一个工作进程建立一个连接后，进程将打开一个文件副本。**所以这个数量还受操作系统设定的，进程最大可打开的文件数有关**。



**3. server匹配规则：**

server_name与host匹配优先级如下：

1、完全匹配

2、通配符在前的，如*.test.com

3、在后的，如[www.test.*](http://www.test.%2A/)

4、正则匹配，如~^\.www\.test\.com$

如果都不匹配

1、优先选择listen配置项后有default或default_server的

2、找到匹配listen**端口**的第一个server块







## 4. 常用命令

window下

start nginx或nginx.exe 启动

nginx -s stop  stop 表示立即停止nginx,不保存相关信息

nginx -s quit quit 表示正常退出nginx,并保存相关信息

nginx -s reload 重启(因为改变了配置,需要重启)

nginx -s reopen 重新打开日志文件

nginx -v 查看Nginx版本：



参考：

[nginx http_core_module](http://nginx.org/en/docs/http/ngx_http_core_module.html)

[nginx中server的匹配顺序](https://www.cnblogs.com/wangzhisdu/p/7839109.html)

[官网负载均衡手册](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)

[nginx documentation](http://nginx.org/en/docs/)

[nginx负载均衡的5种策略（转载）](https://www.cnblogs.com/andashu/p/6377323.html)

[Nginx的一些基本功能极速入门](https://blog.csdn.net/qq_27376871/article/details/52564378)



# 面试题

> #### 为什么不使用多线程？

Apache: 创建多个进程或线程，而每个进程或线程都会为其分配cpu和内存（线程要比进程小的多，所以worker支持比perfork高的并发），并发过大会榨干服务器资源。

Nginx: 采用单线程来异步非阻塞处理请求（管理员可以配置Nginx主进程的工作进程的数量）(epoll)，不会为每个请求分配cpu和内存资源，节省了大量资源，同时也减少了大量的CPU的上下文切换。所以才使得Nginx支持更高的并发。



> #### Nginx是如何处理一个请求的呢？

- 首先，nginx在启动时，会解析配置文件，得到需要监听的端口与ip地址，然后在nginx的master进程里面先初始化好这个监控的socket，再进行listen
- 然后再fork出多个子进程出来,  子进程会竞争accept新的连接。 
- 此时，客户端就可以向nginx发起连接了。当客户端与nginx进行三次握手，与nginx建立好一个连接后。此时，某一个子进程会accept成功，然后创建nginx对连接的封装，即ngx_connection_t结构体
- 接着，根据事件调用相应的事件处理模块，如http模块与客户端进行数据的交换。

- 最后，nginx或客户端来主动关掉连接，到此，一个连接就寿终正寝了



> #### nginx是如何实现高并发的

一个主进程，多个工作进程，每个工作进程可以处理多个请求，每进来一个request，会有一个worker进程去处理。但不是全程的处理，处理到可能发生阻塞的地方，比如向上游（后端）服务器转发request，并等待请求返回。那么，这个处理的worker继续处理其他请求，而一旦上游服务器返回了，就会触发这个事件，worker才会来接手，这个request才会接着往下走。由于web server的工作性质决定了每个request的大部份生命都是在网络传输中，实际上花费在server机器上的时间片不多。这是几个进程就解决高并发的秘密所在。webserver刚好属于网络io密集型应用，不算是计算密集型。



> #### 请解释Nginx如何处理HTTP请求

Nginx使用反应器模式。主事件循环等待操作系统发出准备事件的信号，这样数据就可以从套接字读取，在该实例中读取到缓冲区并进行处理。单个线程可以提供数万个并发连接。

 ?????//todo 看起来不大对，



[8分钟带你深入浅出搞懂Nginx](https://mp.weixin.qq.com/s/IQjB0VV7OjFP8bFgKsgZTg)

[Java面试----2018年Nginx常见面试题](https://blog.csdn.net/wchengsheng/article/details/79930858 )