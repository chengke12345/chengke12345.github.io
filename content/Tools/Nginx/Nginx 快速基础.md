# 1. 并发问题

项目一开始，并发少用户少的时候，一个 tomcat 就够了。
![image|400](Pasted%20image%2020260714135056.png)

但是慢慢的，使用的用户越来越多，并发量慢慢增大，服务器的压力就会变大
![image|400](Pasted%20image%2020260714135343.png)

一个服务器满足不了我们的需求，这时可以增加服务器，多个服务器下，我们就需要一个代理服务器，通过代理服务器来处理请求。
![image|400](Pasted%20image%2020260714135617.png)

代理服务器是一个中间件，代理对其他几台服务器的请求，完成服务请求的转发。用户都访问这个代理服务器就好了。代理服务器自动帮我们维护后面服务器的配置关联。后面三台服务器大小可能不同，比如第一台 64G， 第二台 16G，第三台 8G，因此需要做负载均衡。因此，这个中间件服务器需要两个核心功能，<font color="orange">反向代理</font> 和 <font color="orange">负载均衡</font>。这时候就用到 <font color="orange">Nginx</font>. 

> [!NOTE] 架构的本质：
> 没有什么是加一层解决不了的问题。

# 2. Nginx 简介

Nginx(Engine X), 是一个高性能的 HTTP 和 反向代理 Web服务器软件，也提供了 IMAP/POP3/SMTP 服务。第一个公开版本0.1.0发布于 2004 年10月4日，2011年6月1日发布 nginx 1.0.4 版本。现在最新版本为 nginx 1.30.3。

Nginx 软件的特点是占用内存少，并发能力强，事实上 Nginx 的并发能力在同类型服务器软件中的表现非常突出。Nginx 是一个安装简单，配置文件非常简洁，bug非常少的服务器软件。Nginx 启动特别容易，并且几乎可以做到 7 * 24 小时不间断运行，即使运行数月也不需要重启，还能够在不间断服务的情况下，进行软件升级。

Nginx 代码完全用C语言从头写成，官方测试数据表明，能支持高达 50000个并发连接数的响应。

# 3. 反向代理

首先看一下正向代理
![image|400](Pasted%20image%2020260714160924.png)

VPN 就是正向代理。我们自己无法访问外网，但是在电脑终端，我们可以开一个VPN代理，代理服务器例如在香港，它可以访问外网，它就可以帮我们请求外部资源。我们的终端，请求的是香港服务器，香港服务器再请求美国服务器，美国服务器再返回数据到香港服务器，香港服务器再返回给我们客户端电脑。
这种安装在客户端上的，代理客户端的行为，这种叫做<font color="orange">正向代理</font>。

还有一种代理的服务器端，
![image|400](Pasted%20image%2020260714161818.png)

例如百度网站，它的服务器可能有一些在深圳，有一些在上海，有一些在北京，但是我们永远访问的是baidu.com这个网址。这其实是访问的服务器端的代理。后面无论增删多少服务器，用户是没有感知的，他访问的永远是代理。这种代理就是<font color="orange">反向代理</font>。

<font color="#eb4349">总结：正向代理就是代理的客户端，反向代理就是代理的服务器端</font>。

# 4. 负载均衡

代理服务器后面，真正处理的业务的服务器有的能力大，有的能力小，比如，有64G内存的服务器，也有 16G 内存的服务器。这就涉及到代理服务器需要在其背后的业务服务器上进行负载均衡。Nginx 提供的负载均衡方式有两种，<font color="orange">内置策略</font>和<font color="orange">扩展策略</font>。
内置策略有轮询，加权轮询，IP Hash。扩展策略就天马行空了，只有想不到没有 Nginx 做不到的。

轮询
![image|400](Pasted%20image%2020260714164415.png)

第一个请求给到服务器1，第二个请求给到服务器2，第三个请求给到服务器3。然后，第四个请求给到服务1，第五个给到服务器2 .......以此类推，这样依次循环的，叫轮询。

加权轮询
![image|400](Pasted%20image%2020260714164733.png)

加权轮询，给到服务器一个权重，就会让更多请求给到权重高的服务器，少量的请求给到权重较小的服务器。这样可以保证服务器性能最大化，哪怕有一台很小的服务器，也可以上线使用，它的性能小，负载就小。

IP Hash

IP Hash 对客户端请求的 IP 进行 hash 操作，然后根据 hash 结果将同一个客户端 IP 的请求分发给同一台服务器，可以解决 session 不共享的问题。
![image|400](Pasted%20image%2020260714165553.png)

比如，一个 session 就保存在一个 Tomcat 服务器里面。但是如果我们后面启动了很多台 Tomcat 服务器，就有很多个Tomcat。因此，就必须有 n 个 session。这时候可以用 Redis 做 Session 共享。但是 Nginx 也提供了一种默认算法，就是通过 IP Hash，保证固定 IP 被转发到同一台服务器上，这样 session 就不会丢失，但是服务器效率就大大降低了。现在最常见的方案还是使用 Redis 做 Session 共享。只是，Nginx 提供了这样的方式。

# 5.  动静分离

有些请求是需要后台处理的，有些请求不需要后台处理(比如 CSS，HTML，jpg, js等文件)，这些不需要经过后台处理的文件叫做<font color="orange">静态文件</font>。
让动态网站里的动态网页根据一定规则，把不变的资源和经常变的资源区分开，动静资源做好拆分以后，我们就可以根据静态资源的特点，将其做缓存操作，提高资源响应的速度。

![image | 400](Pasted%20image%2020260714172533.png)


# 6. Windows 下安装 Nginx

Windows 下安装 Nginx 是很简单的，直接下载，解压缩zip。得到如下目录：
![image|600](Pasted%20image%2020260714173357.png)
有一个启动文件 nginx.exe, 有一个conf目录，里面是配置文件。docs 是一些文档。logs 是一些日志文件，temp 是一些临时文件。我们最常使用的就是 conf 目录中的一些配置文件。

其中有一个 nginx.conf 文件。打开里面有一个 `listen 80`, 这是监听端口 80。只要以后访问 80 端口，就会被 nginx 拦截。
启动 nginx.exe, 在浏览器输入 http://localhost/ ，http 的默认端口是 80， 此时如果看到 "Welcome to nginx" 就表明 nginx 启动成功了。

# 7. Linux 下安装 Nginx

同样，在官网下载，是 tar.gz 文件。解压后，看到的目录如下
![image|500](Pasted%20image%2020260714174708.png)

这里面有一个 configure 的执行文件，我们执行它，让它先去自动配置，执行完之后，在该目录下，执行一下 make 命令，再执行 make install 就可以了。
编译完成后，查看 nginx, 就会有一个 nginx 目录 `/usr/local/nginx`, 进入后，就可以看到

![image|400](Pasted%20image%2020260714175221.png)

执行一下 `./sbin/nginx`， 如果没有输出，就表示启动成功了。

# 8.  Nginx 下的常用命令

```bash
cd /usr/local/nginx/sbin/
./nginx              # 启动 nginx  
./nginx -s stop      # 停止
./nginx -s quit      # 安全退出
./nginx -s reload    # 重新加载配置文件
ps aux | grep nginx  # 查看 nginx 进程
```

# 9. nginx.conf 配置文件

Nginx 的配置文件 nginx.conf 有三个部分。最上面的部分是全局配置，这部分配置全局可以生效。第二部分，是 events， 里面可以指定最大连接数，以及指定一些监听事件。第三部分，是 http。
nginx.conf 配置文件的结构其实很简单，基本结构如下：

```nginx.conf
全局配置

events {
	...
}

http {
	http 全局配置
	
	upstream xx {
		// 负载均衡配置
	}
	
	server{
		listen 80;
		server_name localhost;
		// 代理
	}
	
	server{
		listen 443
		server_name localhost;
		// 代理
		
	}
}
```

<font color="deeppink">nginx.conf 配置文件的核心就三个部分，全局配置，events, http</font>。上面就是 nginx.conf 配置文件的基本结构。
全局配置的部分，整个 nginx 都生效。

http 里面有一些 http 的全局配置，比如做一些静态资源的配置和一些小配置等。http 里面有多个 server，不同的 sever 项可以配置不同的服务。80 的请求就会被配置 80 server 的代理处理，443 的请求就会被配置的 443 server 的代理处理。
http 里面还可以做一些流的配置，upstream 可以做一些负载均衡的配置。
平时，主要做 http 配置，性能优化的时候可能需要做一些全局配置。

先看其中一个 server 请求
```nginx
http {
	server{
		listen 80;
		server_name localhost; 
	
		location / {
			// xxx 128.xxx
		}
	
		location /admin {
			// xxx 47.xxx
		}
	}
}
```

我们可以根据访问的资源，把请求转发给不同的服务器处理。
访问 80 端口根目录 `/` 时，就会按照 `location / {...}` 中的配置来执行，比如，它就会交给 128.xxx 这个服务器来处理，而如果访问 `/admin` 它就会转发给 47.xxx 这个服务器处理。这样不同的资源就被区分开了，这是 location 的作用。

以80为例，我们在 location 中，就要配置代理。可以有多台服务器对应处理对 `/` 主页的请求，此时需要先做负载均衡的配置 upstream。
```nginx
http {
	upstream any_name {
		# 服务器资源
		server 127.0.0.1:8080 weight=1；
		server 127.0.0.1:8081 weight=2;
	}
	
	server{
		listen 80;
		server_name localhost; 
	
		location / {
			// xxx 128.xxx
			proxy_pass http://any_name;
		}
	
		location /admin {
			// xxx 47.xxx
		}
	}
}
```

upstream 是做负载均衡的配置，其中配置一些服务器资源。upstream 可以取任意名字，在 location 中配置代理需要使用这个名字。`location /` 中使用 `proxy_pass` 来配置代理，表示任何对根目录 `/` 的请求都代理到 any_name 下面，配置的时候带上协议 http://any_name 。any_name 是一个负载均衡的配置。
upstream 负载均衡中配置 server，比如我们有两台服务器，还可以对服务器设置权重。这个权重值，可以粗略理解为比例，就是 3 次请求中，有 2 次会转发到 8081 服务器上处理，1 次转发到 8080 上处理。
我们通过 location 配置了反向代理，通过 upstream 配置了负载均衡。逻辑上，一个请求到达nginx服务器后，通过 location 找到代理名称，并把请求传递给这个代理，然后再根据代理的负载均衡配置，决定最终把请求传递到哪个物理服务器上处理。