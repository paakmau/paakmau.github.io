+++
title = "Docker下使用Let's Encrypt与nginx搭建支持HTTPS的WordPress博客"
date = 2019-12-18 19:16:13

[taxonomies]
tags = ["Docker"]
categories = ["Docker"]
+++

Let's Encrypt提供免费SSL证书的自动签发续签服务，我们可以用它来为博客启用HTTPS

<!-- more -->

## 签发证书

我们使用certbot/certbot镜像构建一次性容器签发证书

```sh
$ docker volume create data_letsencrypt
$ docker run --rm -p 80:80 -p 443:443 \
   -v data_letsencrypt:/etc/letsencrypt \
   certbot/certbot auth --standalone \
   -m yourname@email.com --agree-tos \
   -d example.com
```

解释：  
\--rm 表示容器结束后就自动删除  
  
其中yourname@email.com为你的邮箱  
example.com为博客的域名  
证书会被生成在data\_letsencrypt数据卷下的live/example.com目录中

## 部署MySQL与WordPress

基本正常部署  
但由于我们使用nginx反向代理，所以WordPress不需要做端口映射

```sh
$ docker volume create data_mysql_wordpress
$ docker volume create data_wordpress
$ docker run -d --name mysql_wordpress \
   -v data_mysql_wordpress:/var/lib/mysql \
   -e MYSQL_ROOT_PASSWORD=root \
   mysql --default-authentication-plugin=mysql_native_password
$ docker run -d --name wordpress \
   -v data_wordpress:/var/www/html \
   --link mysql_wordpress:mysql wordpress
```

## 使用nginx反向代理

```sh
$ docker volume create data_nginx_conf
$ docker run -d --name nginx \
   -v data_nginx_conf:/etc/nginx/conf.d \
   -v data_letsencrypt:/etc/letsencrypt \
   --link wordpress:wordpress \
   -p 80:80 -p 443:443 nginx
```

解释：  
nginx的配置文件会被放在data\_nginx\_conf数据卷中  
映射data\_letsencrypt数据卷以获取证书  
链接wordpress容器用于代理转发

进入data\_nginx\_conf数据卷并修改default.conf如下

```
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
其中ssl\_certificate与ssl\_certificate\_key设定了证书的路径，需要把example.com替换为你博客的域名  
proxy\_pass用于转发wordpress容器的80端口  
最后监听本容器的80端口把HTTP重定向到HTTPS

然后重启nginx容器就完成了

```sh
$ docker restart nginx
```

## 证书续签

证书有效期只有90天，因此需要续签

我们先把nginx停了，把端口让出来

```sh
$ docker stop nginx
```

然后再次使用certbot/certbot进行续签

```sh
$ docker run --rm -p 80:80 -p 443:443 \
   -v data_letsencrypt:/etc/letsencrypt \
   certbot/certbot renew --standalone
```

之后再启动nginx就行了

```sh
$ docker start nginx
```

然后可以用crontab创建定时任务自动续签
