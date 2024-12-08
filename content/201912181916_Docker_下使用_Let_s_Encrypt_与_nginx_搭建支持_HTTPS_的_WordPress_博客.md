+++
title = "Docker 下使用 Let's Encrypt 与 nginx 搭建支持 HTTPS 的 WordPress 博客"
date = 2019-12-18 19:16:13
slug = "201912181916"

[taxonomies]
tags = ["Docker", "Let's Encrypt", "nginx", "HTTPS", "WordPress"]
+++

Let's Encrypt 提供免费 SSL 证书的自动签发续签服务，我们可以用它来为博客启用 HTTPS

<!-- more -->

## 签发证书

我们使用 `certbot/certbot` 镜像构建一次性容器签发证书

```sh
docker volume create data_letsencrypt
docker run --rm -p 80:80 -p 443:443 \
  -v data_letsencrypt:/etc/letsencrypt \
  certbot/certbot auth --standalone \
  -m yourname@email.com --agree-tos \
  -d example.com
```

解释：

- `--rm` 表示容器结束后就自动删除。
- 证书会被生成在 `data_letsencrypt` 数据卷下的 `live/example.com` 目录中。

## 部署 MySQL 与 WordPress

基本就正常部署就行<br>
但由于我们使用 nginx 反向代理，所以 WordPress 不需要做端口映射

```sh
docker volume create data_mysql_wordpress
docker volume create data_wordpress
docker run -d --name mysql_wordpress \
  -v data_mysql_wordpress:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=root \
  mysql --default-authentication-plugin=mysql_native_password
docker run -d --name wordpress \
  -v data_wordpress:/var/www/html \
  --link mysql_wordpress:mysql wordpress
```

## 使用 nginx 反向代理

```sh
docker volume create data_nginx_conf
docker run -d --name nginx \
  -v data_nginx_conf:/etc/nginx/conf.d \
  -v data_letsencrypt:/etc/letsencrypt \
  --link wordpress:wordpress \
  -p 80:80 -p 443:443 nginx
```

解释：

- nginx 的配置文件会被放在 `data_nginx_conf` 数据卷中
- 映射 `data_letsencrypt` 数据卷以获取证书
- 链接 `wordpress` 容器用于代理转发

进入 `data_nginx_conf` 数据卷并修改 `default.conf` 如下

```conf
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;


    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass http://wordpress:80;
    }
}

server {
    listen 80;
    return 301 https://$host$request_uri;
}
```

解释：

- 其中 `ssl_certificate` 与 `ssl_certificate_key` 设定了证书的路径，需要把 example.com 替换为你博客的域名
- `proxy_pass` 用于转发 `wordpress` 容器的 80 端口。最后监听本容器的 80 端口把 HTTP 重定向到 HTTPS

然后重启 `nginx` 容器就完成了

```sh
docker restart nginx
```

## 证书续签

证书有效期只有 90 天，因此需要续签

我们先把 `nginx` 容器停了，把端口让出来

```sh
docker stop nginx
```

然后再次使用 `certbot/certbot` 进行续签

```sh
docker run --rm -p 80:80 -p 443:443 \
  -v data_letsencrypt:/etc/letsencrypt \
  certbot/certbot renew --standalone
```

之后再启动 `nginx` 就行了

```sh
docker start nginx
```

然后可以用 `crontab` 创建定时任务自动续签
