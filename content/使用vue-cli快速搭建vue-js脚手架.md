+++
title = "使用Vue CLI快速搭建Vue.js脚手架"
date = 2019-12-27 10:24:33

[taxonomies]
tags = ["Vue.js"]
categories = ["Node.js"]
+++

本文基于  

<!-- more -->
Vue CLI 4.1.2  
Node.js v13.5.0

## 简单介绍

在Vue框架中，我们可以通过数据驱动视图，实现数据与视图的分离  
从而将关注点集中在逻辑上，让框架自动替我们完成麻烦的数据绑定

同时它也是一个容易上手的渐进式框架

## 创建Vue.js项目

用npx命令运行Vue CLI  
它能提供一些选项帮助我们建立脚手架项目

```sh
$ npx @vue/cli create hello-world
```

这里npx会先下载再运行Vue CLI，不会污染全局依赖也不会在当前目录留下任何东西，十分干净

本文选择的配置如下

![](https://hebomou.top/wp-content/uploads/2020/01/image-1024x221.png)

Please pick a preset  
我们选择Manually select features以手动选择需要的特性  
当然也可以使用自己定义的预设

Check the features needed for your project  
本文选了几个常用的  
Babel：能够把ES6版本的代码转换为向后兼容的JavaScript语法  
Router：官方提供的路由管理器，用于实现单页应用  
Vuex：官方提供的状态管理器，用于解决组件之间传递数据，共享数据的问题  
CSS Pre-processors：CSS预处理器，可以为CSS添加一些编程特性，方便CSS的编写，最后发布的时候再重新编译为浏览器可识别的CSS代码  
Linter / Formatter：用于语法和格式检查  
Unit Testing：用于单元测试

Use history mode for router  
历史模式，这能把URL中的#去掉。不管它，开就完事了

Pick a CSS pre-processor  
CSS预处理器，使用Less。哪个都行，与本文无关

Pick a linter / formatter config  
看个人喜好，然后选择保存时检查

Pick a unit testing solution  
单元测试我们选Jest

Where do you prefer placing config  
插件配置我喜欢整合起来，统一放在package.json里

Save this as a preset  
不需要保存为预设

Pick the package manger  
我用的npm

然后它会把项目生成在当前目录下的hello-world文件夹中

## 运行

进入项目文件夹并在终端中输入

```sh
$ npm run serve
```

得到这样的输出

![](https://hebomou.top/wp-content/uploads/2020/01/image-2.png)

然后在浏览器中打开就行了

## 简单的项目介绍

这个项目大概长这样

![](https://hebomou.top/wp-content/uploads/2020/01/image-1-351x1024.png)

assets  
放些图片什么的

components  
存放可复用的组件

router  
在这里为单页应用配置路由

store  
Vuex相关，用于全局的状态管理

views  
存放网站的所有页面

tests  
单元测试

main.js  
这里初始化Vue.js，可以加载全局插件、全局组件

package.json  
用于配置Node.js项目。然后因为我们在CLI中的配置，所以ESLint、Jest这些东西的配置也在里面
