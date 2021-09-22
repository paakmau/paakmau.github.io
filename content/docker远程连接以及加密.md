+++
title = "Docker远程连接以及TLS加密"
date = 2019-12-11 21:26:36

[taxonomies]
tags = ["Docker"]
categories = ["Docker"]
+++

Docker Daemon运行在Debian上，Docker Client在macOS上

<!-- more -->

因为不想在本地安装Docker，考虑在本地用Docker Client远程连接到远程服务器的Docker Daemon上。

## 名词解释

Docker Daemon，可以理解为Docker的后端，也是Docker项目实际运行的地方。

Docker client，Docker的客户端，可以用来控制Daemon运行Docker项目。

## 服务端（Debian）配置远程连接

覆盖配置docker.service即可

修改/etc/systemd/system/docker.service.d/override.conf的内容如下  
（如果目录或文件不存在就直接创建）

```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2337 -H unix:///var/run/docker.sock
```

解释：  
第二行用于清除原本的ExecStart，不可缺少  
  
\-H tcp://0.0.0.0:2337  
监听2337端口，使得Daemon可以由2337端口远程访问  
  
\-H unix:///var/run/docker.sock  
监听Unix域Socket，使得原本的本地连接方式依然可用

重启后台进程与Docker服务

```sh
$ systemctl daemon-reload
$ systemctl restart docker
```

至此，我们就能通过2337端口远程访问Docker Daemon了

## 客户端（macOS）配置远程连接

安装Docker Client

```sh
$ brew install docker
```

临时指定Docker服务器的地址  
需要修改x.x.x.x为服务器的ip  
如有需要，可以把DOCKER\_HOST环境变量写入.bashrc或者.zshrc

```sh
$ export DOCKER_HOST=tcp://x.x.x.x:2337
```

显示Docker信息以确认是否成功

```sh
$ docker info
```

## 服务端（Debian）配置远程连接加密

发现远程连接不需要验证，如果有人获取了我们服务器的ip与端口，他就能直接连接到Docker Daemon。并且Docker具有root权限，不加密十分危险。

所以接下来我们快速配置加密连接

执行下述脚本在/etc/docker目录下直接创建密钥

```sh
# 创建certs目录并切换路径
mkdir /etc/docker/certs
cd /etc/docker/certs

# 创建ca密钥
openssl genrsa -aes256 -out ca-key.pem 4096

# 生成ca证书
openssl req -new -x509 -days 1000 -key ca-key.pem -sha256 -subj "/CN=*" -out ca.pem

# 创建服务器私钥
openssl genrsa -out server-key.pem 4096

# 生成服务器证书
openssl req -subj "/CN=*" -sha256 -new -key server-key.pem -out server.csr

# 使用ca证书与服务器证书签名
openssl x509 -req -days 1000 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem

# 创建客户端私钥
openssl genrsa -out key.pem 4096

# 生成客户端证书
openssl req -subj "/CN=client" -new -key key.pem -out client.csr

# 创建并写入配置文件，然后根据配置文件使用ca证书与客户端证书签名
echo extendedKeyUsage=clientAuth > extfile.cnf
openssl x509 -req -days 1000 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out cert.pem -extfile extfile.cnf

# 删除多余的文件
rm -rf ca.srl client.csr extfile.cnf server.csr
```

接下来覆盖配置docker.service  
修改/etc/systemd/system/docker.service.d/override.conf的内容如下  
（如果目录或文件不存在就直接创建）

```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --tlsverify --tlscacert=/etc/docker/certs/ca.pem --tlscert=/etc/docker/certs/server-cert.pem --tlskey=/etc/docker/certs/server-key.pem -H tcp://0.0.0.0:2337 -H unix:///var/run/docker.sock
```

解释：基本跟上文的远程连接配置相同，只是指定了tls认证方式

## 客户端（macOS）配置远程连接加密

首先拷贝密钥  
将服务器上/etc/docker/certs下的ca.pem、cert.pem、key.pem拷贝到客户端的~/.docker中

然后配置Docker环境变量到.zshrc或.bashrc中

```sh
export DOCKER_HOST=tcp://x.x.x.x:2337 DOCKER_TLS_VERIFY=1
```

尝试查看Docker信息以确认是否成功

```sh
$ docker info
```

## 参考资料

[https://docs.docker.com/engine/security/https/](https://docs.docker.com/engine/security/https/)  
[https://www.jianshu.com/p/9e513f57853b](https://www.jianshu.com/p/9e513f57853b)
