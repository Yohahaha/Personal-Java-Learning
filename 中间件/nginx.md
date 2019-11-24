#### 命令

> nginx // 启动

> nginx -s [signal]

- stop --- 快速关闭
- quit --- 优雅关闭
- reload --- 重新加载配置文件

#### 配置文件

Nginx默认配置文件位于`/etc/nginx/nginx.conf`，每次对其修改后都可以通过`nginx -s reload`热加载配置文件。

初始内容：

```

user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
	# 下面这两个是加载一些默认配置，比如nginx欢迎页
    # include       /etc/nginx/mime.types;
    # default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

	# 这里会去包含该目录下的default.conf
    # include /etc/nginx/conf.d/*.conf;

}

```

##### 静态资源映射

```
server {
    location / {
        root /data/www;
    }

    location /images/ {
        root /data;
    }
}
```

将所有以`/images/`开始的URI映射到本地文件系统的`/data/images/`目录下，如`localhost/images/1.jpg`请求会访问`/data/images/1.jpg`文件。

将其他所有URI映射到`/data/www/`目录下，如`localhost/example.html`请求会访问`/data/www/example.html`文件。

##### 代理

假设在本地8080端口上启动一个服务，下面配置会将请求转发到该服务上

```
server {
    location / {
        proxy_pass http://localhost:8080;
    }
}
```

