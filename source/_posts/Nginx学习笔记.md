---
title: Nginx学习笔记
mathjax: true
date: 2023/4/19
category: 技术学习与分享
---


# Nginx学习笔记

## 一、一些概念

### 1、正向代理以及反向代理

#### 是什么

​	在资源的访问过程中通过代理服务器，正向代理服务器接收客户端的请求并将其发给服务器，此时代理服务器是一个客户端；反向代理接收客户端的请求并将其发给服务器，此时代理服务器是一个服务端

#### 为什么

​	正向代理可以突破IP的访问限制、提高访问速度等；而反向代理一大作用是可以实现负载均衡

​	负载均衡：反向代理服务器将请求平均分配到自己代理的服务器上，避免某些服务器负载过重（*而且这样的话多台服务器就可以共用一个对外的IP了）

### 2、Nginx

​	Nginx 是高性能的 HTTP 和反向代理的web服务器,可以作为静态页面的web服务器，结合CGI协议的动态语言可以实现动态页面

## 二、Nginx常用命令

前提：进入nginx生成的sbin目录下

```bash
./nginx -v#启动nginx
./nginx -s stop#关闭nginx
./nginx -s reload#重新加载nginx(不是重启服务器，只是重新编译？)
```



## 三、Nginx的配置文件

### 关于位置

linux用apt安装的nginx core的配置文件在*/etc/nginx.conf*

在这个nginx.conf的***http块***（见下文）里面有这样两句

```Ngin
        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
```

第一句可以从conf.d里导入所有后缀名为.conf的文件，感觉新增加server之类的字段应该弄一个文件放里面？(那sites-enabled里应该放什么？)

include大概就是直接把文件内容扔到自身所在的配置文件里面（自己写配置文件的时候应该也可以这样优化结构）

### 配置文件的组成

#### Part1 全局块

主要设置一些影响nginx服务器整体运行的配置指令

#### Part2 events块

主要影响nginx服务器与用户的网络连接（配置连接属性be like），对服务器的性能影响较大

#### Part3 http块

​	大多数功能的定义和第三方模块的配置

##### --http全局块

##### --server块

​	每个 http 块可以包括多个 server 块，而**每个 server 块就相当于一个虚拟主机**。
​	而每个 server 块也分为全局 server 块，以及可以同时包含多个 locaton 块。

​	全局server块主要负责指定虚拟主机监听配置、名称、IP配置等

​	Location块主要负责对传入的地址进行处理（某种意义上是个定制化的router？）

**一个简单的 server块示例**

```nginx
 server{
                listen 81;
                server_name localhost;
                location /{
                    root html;
                    index index.html index.htm;
        `			#这个应该是nginx的默认页面
                }


        }
```

上面的server块添加到htttp块之后，就可以通过访问81端口获取页面了

**在server块里面进行代理设置**

```nginx
server{
    listen 81;
    server_name localhost;
    location /{
        root html;
        proxy_pass https://baidu.com
    }
}
```

这样，用户在访问localhost:81的时候就可以跳转到baidu.com了

但是，如果想要把localhost换成其他的地址的话，似乎要配置一下本机的域名解析之类的东西

还有，proxy_pass不能缺少通信协议的前缀

代理的逻辑：

```
proxy_pass没有上下文的时候：
例如：proxy_pass https://baidu.com; 
你的请求：
http://{server_name}:{port}/{uri}

nginx代理后：
http://{proxy_pass}/{uri}

proxy_pass有上下文（proxy_uri）时
例如
proxy_pass https://baidu.com/; 
你的请求：
http://{server_name}:{port}/{uri}

nginx代理后：
http://{proxy_pass}/{proxy_uri}+{uri减去location匹配的部分}


```

location通过匹配你的uri是否含有某个特定路径（通过正则表达式或者直接严格匹配）来决定要设置哪一个代理，所以，在请求服务的时候应当像直接向服务器发起请求一样写地址（或者说，nginx并不会处理请求，而是将host改变之后将uri原封不动传给服务器）

1.proxy_pass的目标地址，默认不带/，表示只代理域名，url和参数部分不会变（把请求的path拼接到proxy_pass目标域名之后作为代理的URL）

2.如果在目标地址后增加/，则表示把path中location匹配成功的部分剪切掉之后再拼接到proxy_pass目标地址

**location的正则匹配**(仅支持proxy_server没有上下文的时候)

> 该指令用于匹配 URL。
> 语法如下：
>
> 1、= ：用于不含正则表达式的 uri 前，要求请求字符串与 uri 严格匹配
> 2、~：用于表示 uri 包含正则表达式，区分大小写。
> 3、~*：用于表示 uri 包含正则表达式，不区分大小写。
> 4、^~：用于不含正则表达式的 uri 前，要求 Nginx 服务器找到标识 uri 和请求字 符串匹配度最高的 location 后，立即使用此 location 处理请求，而不再使用 location 块中的正则 uri 和请求字符串做匹配。

#### location的一些参数的作用

```
root 
alias
proxy_pass
```

## 四、nginx docker的使用

### docker中的一些目录地址以及挂载方法

```bash
/usr/share/nginx/html #容器根目录
/etc/nginx/(conf) #配置文件位置

```

#### 可以将宿主机的目录挂载到容器里面

```bash
$ docker run --name mynginx2 \
   --mount type=bind,source=/path/to/source,target=/usr/share/nginx/html,readonly \
   --mount type=bind,source=/path/to/conf,target=/etc/nginx/conf,readonly \
   -p 80:80 \
   -d nginx
```

#### 可以将内容以及配置文件复制到配置目录里面

```dockerfile
FROM nginx
RUN rm /etc/nginx/conf.d/default.conf
COPY /path/to/content /usr/share/nginx/html
COPY /path/to/conf /etc/nginx/conf.d
```

#### 还可以通过一个helper container实现在docker里面修改文件

第一步：创建nginx镜像并运行

```dockerfile
#dockerfile
FROM nginx
COPY content /usr/share/nginx/html
COPY conf /etc/nginx
VOLUME /usr/share/nginx/html
VOLUME /etc/nginx
```

第二步：创建容器

```bash
docker build -t mynginx_image2 .
```

第三步：创建一个helper container

```bash
docker run --name mynginx4 -p 80:80 -d mynginx_image2
```

第四步：运行helper container

```
docker run -i -t --volumes-from mynginx4 --name mynginx4_files debian /bin/bash
```

然后就可以进入debian的终端了

后面要再次使用的话

```bash
docker start mynginx4_files
docker stop mynginx4_files
```

### 使用docker控制nginx

nginx本身没有shell access，要重新加载docker里面的nginx的话，需要用docker kill命令给容器传输一个HUP信号

```bash
docker kill -s HUP container-name
```

如果要重启nginx的话，直接重启容器就好

```bash
docker restart container-name
```





## 参考

[Beginner’s Guide (nginx.org)](http://nginx.org/en/docs/beginners_guide.html)

[nginx 反向代理 URL替换方案_nginx替换url参数_皮卡车厘子的博客-CSDN博客](https://blog.csdn.net/yk614294861/article/details/102688926)

[Deploying NGINX and NGINX Plus on Docker | NGINX Plus](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-docker/)