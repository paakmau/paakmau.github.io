+++
title = "Express + MySQL + Session搭建Web后端脚手架"
date = 2019-12-18 00:59:09
slug = "201912180059"

[taxonomies]
tags = ["MySQL", "Node.js", "Express"]
categories = ["Node.js"]
+++

因为是我某门课的作业才顺便写的，因为感觉js后端偏向玩具所以不太感兴趣。但还是不得不说这玩具真好玩，贼牛逼。Express框架用几十行代码就能搭出几乎完整的后端脚手架

<!-- more -->

本项目是一个近似完整的后端脚手  
以Express框架为基础，以MySQL作为数据存储，有跨域处理，有Session。

但是该脚手架缺少Express异常处理，感兴趣可以自行添加  
而且用MySQL存Session感觉不太正常，应该换Redis，篇幅有限就不写了（虽然已经很长了）

作为示例，本文还实现了基于Session的验证码认证、RESTful风格接口与MD5加密的登陆注册

## 环境

Node.js v13.3.0

MySQL 8.0.18，监听3306端口，认证插件为mysql\_native\_password  
root账户的密码为root

在MySQL中执行以下脚本建立两个数据库，分别用于存储Session数据与用户数据

```SQL
CREATE SCHEMA `hello_session` ;
CREATE SCHEMA `hello` ;

USE `hello`;
CREATE TABLE `hello`.`user` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `username` VARCHAR(45) NOT NULL,
  `password` VARCHAR(45) NOT NULL,
  PRIMARY KEY (`id`));
```

## 新建Node.js项目

然后建立文件夹并初始化Node.js项目

```sh
$ mkdir hello-world
$ cd hello-world
$ npm init
```

然后一路回车即可  
本文配置中入口点（entry point）使用了默认值，即index.js

## Express框架快速上手

首先安装express依赖

```sh
$ npm install express
```

然后修改入口点index.js代码如下

```js
const express = require("express")
const app = express()

// 定义一个Get请求的回调
app.get('/helloworld', (req, res) => res.send("hello world!"))

// 将后端开在8080端口
app.listen(8080)
```

于是我们测试一下  
在终端中输入如下命令即可运行index.js

```sh
$ node index.js
```

在Postman中对localhost:8080/helloworld发送Get请求，得到如下结果

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191217-005416@2x.png)

Express框架就搭建成功了

## Express中间件

Express每次处理Request都会经过所有的中间件，可以想像成流水线。因此可以利用它为Express添加新的功能

而对于上文代码中的app，调用app.use方法就能为app添加中间件

## 引入body-parser中间件

这个中间件能将HTTP请求体中的JSON转换为js对象属性

首先安装body-parser依赖

```sh
$ npm install body-parser
```

接下来我们测试一下获取body的内容  
修改index.js如下，并运行

```js
const express = require("express")
const app = express()

// 添加body-parser
var bodyParser = require("body-parser")
app.use(bodyParser.json())
app.use(bodyParser.urlencoded({ extended: false }))

// 添加post请求并获取body的json数据
app.post('/helloworld', (req, res) => res.send("You msg is: " + req.body.msg));

app.listen(8080)
```

Postman中发送post请求，并收到回应，见下图。

![](https://hebomou.top/wp-content/uploads/2019/12/QQ20191217-011755@2x-1024x505.png)

## 引入express-session中间件

Session是一种存储在后端的缓存数据，可以理解为以id索引的一块临时数据。前端将id存储在Cookie中，它把id传递给后端，后端就能根据id找到对应的Session。它的安全性相对Cookie高很多，一般用于登陆、验证码等。  
Session可以简单存储在内存中，也可以存储在数据库中

在express中可以通过express-session中间件实现Session功能

为了方便我们选择mysql作为Session的存储方式（但一般使用redis数据库）

它会使用我们最开始时建立的mysql数据库hello\_session  
（express-session会自动建表所以不用担心）

然后测试一下看看  
修改index.js如下，并运行

```js
const express = require("express")
const app = express()

// 添加express-session
var session = require("express-session")
var mysqlStore = require("express-mysql-session")
app.use(session({
    store: new mysqlStore({
        host: 'localhost',
        port: 3306,
        user: 'root',
        password: 'root',
        database: 'hello_session'
    }),
    secret: 'hello-secret',
    cookie: {
        maxAge: 1800000
    },
}))

// 设置session.hello
app.get('/set-session', (req, res) => {
    req.session.hello = "Hello!"
    res.send("session.hello has been set.")
})
// 获取session.hello的值
app.get('/get-session', (req, res) => {
    if (req.session.hello == undefined)
        res.send("session.hello is undefined.")
    else
        res.send(req.session.hello)
})

app.listen(8080)
```

解释：  
对于同一个客户端，req.session是相同的，它其实是根据客户端持有的Session ID（如果没有，则会自动分配一个）找到的对应Session

Postman中直接请求get-session会收到“session.hello is undefined.”，但是如果先请求set-session在get-session就会收到“Hello!”

## 跨域处理

因为前后端分离，前端与后端会部署在不同的服务器或者同一台服务器的不同端口上，所以前端要访问后端接口就会产生跨域问题。

而由于使用了Session，前端需要将Cookie的内容传递给后端，跨域配置的安全性要求也就更高

可以参考后文代码中配置示例

## 生成验证码

安装svg-captcha库

```sh
$ npm install svg-captcha
```

尝试生成验证码  
修改index.js如下并运行

```js
const svgCaptcha = require("svg-captcha")
let captcha = svgCaptcha.create({
        // 翻转颜色
        inverse: false,
        // 字体大小
        fontSize: 36,
        // 噪声线条数
        noise: 3,
        // 宽度
        width: 80,
        // 高度
        height: 30,
    })
console.log(captcha.text)
console.log(captcha.data)
```

解释：  
svg-captcha的create方法生成一个验证码对象  
它的text属性是验证码文本，data属性是一个svg图像

## MD5加密

MD5加密其实就是一个哈希函数，只是他的哈希值长的又丑又长，不可逆  
因此把用户的密码用MD5算法哈希后存储，就可以通过哈希值比对用户密码实现登陆，也没有人能知道密码原文

安装utility依赖

```sh
$ npm install utility
```

尝试加密  
修改index.js如下，并运行

```js
const utility = require("utility")
function encodePassword(pwd) {
    return utility.md5(pwd + "hello express md5")
}

console.log(encodePassword("test_password"))
```

得到如下密文  
```
9c6481dbc7c568e6e0ed338c415db921
```

## 连接MySQL

这部分与Express无关

这里会使用到我们最初建立的hello数据库及其中的user表

尝试连接MySQL服务器，并添加一条测试记录  
编辑index.js脚本如下

```js
const mysql = require("mysql")

// 建立连接
const mysqlConnect = mysql.createConnection({
    host: 'localhost',
    port: 3306,
    user: 'root',
    password: 'root',
    database: 'hello'
})
mysqlConnect.connect()

// 添加测试记录
 mysqlConnect.query("INSERT INTO `web_homework`.`user` (`username`, `password`) VALUES (?, ?);", ["test_username", "test_password"], (err, result, field) => { if (err) console.log(err) })
```

## 构造完整的脚手架

编辑index.js如下并运行

```js
const express = require("express")
const app = express()

// 跨域
app.all("*", function (req, res, next) {
    res.header("Access-Control-Allow-Credentials", "true");
    res.header("Access-Control-Allow-Origin", "null");
    res.header("Access-Control-Allow-Headers", "Content-Type, Content-Length, Authorization, Accept,X-Requested-With");
    res.header("Access-Control-Allow-Methods", "PUT, POST, GET, DELETE, OPTIONS");
    res.header("X-Powered-By", '3.2.1');
    res.header("Content-Type", "application/json; charset=utf-8");
    next();
});

// 添加body-parser中间件
const bodyParser = require("body-parser")
app.use(bodyParser.json())
app.use(bodyParser.urlencoded({ extended: false }))

// 添加session中间件
const session = require("express-session")
const mysqlStore = require("express-mysql-session")
app.use(session({
    store: new mysqlStore({
        host: 'localhost',
        port: 3306,
        user: 'root',
        password: 'root',
        database: 'hello_session'
    }),
    secret: 'web-homework-2019',
    resave: false,
    saveUninitialized: true,
    cookie: {
        maxAge: 1800000,
        secure: false
    },
}))

// 验证码库
const svgCaptcha = require("svg-captcha")
// MD5加密
const utility = require("utility")
function encodePassword(pwd) {
    return utility.md5(pwd + "hello express md5")
}

console.log(encodePassword("test_password"))

// mysql配置
const mysql = require("mysql")
const mysqlConnect = mysql.createConnection({
    host: 'localhost',
    port: 3306,
    user: 'root',
    password: 'root',
    database: 'hello'
})
mysqlConnect.connect()

// 注册
app.post("/user", (req, res, next) => {
    let username = req.body.username
    let password = req.body.password
    let captcha = req.body.captcha
    // 验证输入合法
    if (username.length < 6  password.length < 6) { res.send({ success: false, data: "账号密码都应该大于等于六位" }); return }
    // 认证验证码
    if (req.session.captcha == undefined  captcha.toLowerCase() != req.session.captcha.toLowerCase()) { res.send({ success: false, data: "验证码错误" }); return }
    // MD5并尝试插入
    password = encodePassword(password)
    mysqlConnect.query('SELECT id FROM `hello`.`user` WHERE `username` = ?', [username], (err, result, field) => {
        if (err) { res.send({ success: false, data: err }); return }
        if (result.length != 0) { res.send({ success: false, data: "账户已存在" }); return }
        mysqlConnect.query("INSERT INTO `hello`.`user` (`username`, `password`) VALUES (?, ?);", [username, password], (err, result, field) => {
            if (err) { res.send({ success: false, data: err }); return }
            else
                res.send({ success: true })
        })
    })
})

// 登陆
app.post("/token", (req, res, next) => {
    let username = req.body.username
    let password = req.body.password
    let captcha = req.body.captcha
    // 验证输入合法
    if (username.length < 6  password.length < 6) { res.send({ success: false, data: "账号密码都应该大于等于六位" }); return }
    // 认证验证码
    if (req.session.captcha == undefined  captcha.toLowerCase() != req.session.captcha.toLowerCase()) { res.send({ success: false, data: "验证码错误" }); return }
    // MD5并验证
    password = encodePassword(password)
    mysqlConnect.query('SELECT password FROM `hello`.`user` WHERE `username` = ?', [username], (err, result, field) => {
        if (err) { res.send({ success: true, data: err }); return }
        if (result.length == 0) { res.send({ success: false, data: "账户名不存在" }); return }
        if (result[0].password != password) { res.send({ success: false, data: "密码错误" }); return }
        req.session.username = username
        res.send({ success: true })
    })
})

// 注销
app.delete("/token", (req, res, next) => {
    req.session.username = undefined
    res.send({ success: true })
})

// 获取验证码接口
app.get("/captcha", (req, res) => {
    let captcha = svgCaptcha.create({
        // 翻转颜色
        inverse: false,
        // 字体大小
        fontSize: 36,
        // 噪声线条数
        noise: 3,
        // 宽度
        width: 80,
        // 高度
        height: 30,
    })
    req.session.captcha = captcha.text;
    res.type('svg');
    res.send({ success: true, data: captcha.data });
})

// 获取用户名
app.get("/username", (req, res) => {
    if (req.session.username == undefined)
        res.send({ success: false, data: "尚未登陆" })
    else
        res.send({ success: true, data: req.session.username })
})

app.listen(8080)
```

注释很详细，结合前文的科普应该不难理解了
