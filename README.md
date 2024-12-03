
## 1 介绍


记录使用docker 构建包含 `supervior` 的镜像,


`supervisor`: 是一个管理和监控进程的程序,可以方便的通过配置文件来管理我们的任务脚本


将supervisor构建到系统镜像中,启动时默认启动 supervisor进程,容器可以正常运行,然后通过 ,supervisor 去启动,停止,重启我们的 任务脚本,例如, flask, fastapi 等服务启动脚本


### 1\.1 背景


构建一个 `fastapi web` 服务镜像,方便业务上线.基础需求:


1. 容器中没有 `fastapi web` 服务也可以正常启动运行
2. `web 服务`通过配置文件可插拔加载和分离


## 2 具体流程


### 2\.1 准备supervisor默认启动的配置文件


使用默认的配置文件可以 默认开启 supervisor 的 http 服务器管理访问功能,即配置块 `inet_http_server`


对应的配置如下:



```
; supervisor config file

[unix_http_server]
file=/var/run/supervisor.sock   ; (the path to the socket file)
chmod=0700                       ; sockef file mode (default 0700)

[supervisord]
nodaemon=true
logfile=/var/log/supervisor/supervisord.log ; (main log file;default $CWD/supervisord.log)
pidfile=/var/run/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
childlogdir=/var/log/supervisor            ; ('AUTO' child log dir, default $TEMP)

; the below section must remain in the config file for RPC
; (supervisorctl/web interface) to work, additional interfaces may be
; added by defining them in separate rpcinterface: sections
[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

# [supervisorctl]
# serverurl=unix:///var/run/supervisor.sock ; use a unix:// URL  for a unix socket

; The [include] section can just contain the "files" setting.  This
; setting can list multiple files (separated by whitespace or
; newlines).  It can also contain wildcards.  The filenames are
; interpreted as relative to this file.  Included files *cannot*
; include files themselves.

[include]
files = /etc/supervisor/conf.d/*.conf


[inet_http_server]         ; inet (TCP) server disabled by default
port=*:9001        ; (ip_address:port specifier, *:port for all iface)
username=admin              ; (default is no username (open server))
password=admin.123              ; (default is no password (open server))


```

### 2\.2 web服务的pip包安装配置文件


准备 `requirements.txt`, 使用时根据自己实际情况.



```
fastapi==0.115.0

```

### 2\.3 Dockerfile构建配置


这里我使用 `python:3.10-slim` 作为基础镜像构建, supervisor的启动命令作为入口 `CMD`



```
FROM python:3.10-slim 

MAINTAINER faron
WORKDIR /usr/src/app

RUN mkdir -p /var/log/supervisor \
    && apt-get update && apt-get install -y cron autoconf automake libtool vim procps supervisor \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* 

COPY requirements.txt ./
COPY supervisord.conf /etc/supervisor/supervisord.conf
RUN pip install -i https://mirrors.aliyun.com/pypi/simple --no-cache-dir -r requirements.txt \
    && rm -rf requirements.txt

CMD ["/usr/bin/supervisord","-c","/etc/supervisor/supervisord.conf" ]

```

构建命令:



```
docker build -t web:1.0 .

```

### 2\.4 启动测试


使用 `docker-compose` 进行测试



```
services:
  web: # web服务
    image: web:1.0
    container_name: web_test
    ports:
      - 9001:9001
    volumes:
      - /etc/localtime:/etc/localtime
    restart: always


```

启动命令:



```
docker-compose up -d

```


```
NAME       IMAGE     COMMAND                  SERVICE   CREATED         STATUS         PORTS
web_test   web:1.0   "/usr/bin/supervisor…"   web       8 seconds ago   Up 7 seconds   0.0.0.0:9001->9001/tcp

```

**查看效果**: 访问 IP:9001 ,账户名密码: admin admin.123
账户名密码在 `supervisord.conf` 文件的 `inet_http_server` 下;


![](https://1blog1.oss-cn-hangzhou.aliyuncs.com/blogs/202412022028264.png)


## 3 使用supervisor配置启动服务


这里以一个 实时查看文本文件的需求服务为示例,介绍如何通过 supervisor 管理服务;


### 3\.1 创建 服务配置文件


实时查看某个文件的服务管理脚本: test.conf



```
[program:test]
command=tail -n 20 -f /var/log/supervisor/supervisord.log
autostart=true
startsecs=3
autorestart=true


```

调整`docker-compose.yml`文件,将启动配置文件映射进去



```
services:
  web: # web服务
    image: web:1.0
    container_name: web_test
    ports:
      - 9001:9001
    volumes:
      - /etc/localtime:/etc/localtime
      - ./test.conf:/etc/supervisor/conf.d/test.conf  # 映射启动配置文件
    restart: always

```

启动新的容器服务,注意历史的关闭



```
docker-compose down
docker-compose up -d

```

访问 IP:9001


![](https://1blog1.oss-cn-hangzhou.aliyuncs.com/blogs/202412022057882.png)


## 4 参考文档


* [supervisor configuration](https://github.com)
* [supervisor program config](https://github.com):[Flowercloud 机场订阅加速](https://flowercloud6.com)


