+++
title = "Vue Router入门学习笔记"
date = 2020-01-17 20:06:02

[taxonomies]
tags = ["Vue.js"]
categories = ["Node.js"]
+++

Vue Router是Vue.js官方提供的路由管理器，轻松构建单页应用

<!-- more -->

本文基于用Vue CLI创建的脚手架，包括动态路径参数、嵌套路由、命名视图、懒加载

## 引入Vue Router

在使用Vue CLI构建时需要引入Router

如果没有，需要安装vue-router依赖，然后修改main.js等，但很麻烦就懒得写了

## 脚手架中的相关文件

先介绍一下脚手架中与路由相关的文件

src/router/index.js  
router的配置文件，管理路由。这里展示一个简易版的

```js
import Vue from "vue";
import VueRouter from "vue-router";
import Home from "../views/Home.vue";
import About from "../views/About.vue";

Vue.use(VueRouter);

const routes = [
  {
    path: "/",
    name: "home",
    component: Home
  },
  {
    path: "/about",
    name: "about",
    component: About
  }
];

const router = new VueRouter({
  mode: "history",
  base: process.env.BASE_URL,
  routes
});
export default router;
```

解释：  
routes规定了路由与组件之间的映射，页面可以import进来  
router是路由管理器，这里有路由模式、基地址的配置，并把routes传入进来

src/App.vue  
它是单页应用的入口（我不知道怎么形容，总之就是所有组件都会渲染在这个页面中），贴一下关键部分

```html
<template>
  <div id="app">
    <div id="nav">
      <router-link to="/">Home</router-link> 
      <router-link to="/about">About</router-link>
    </div>
    <router-view />
  </div>
</template>
```

解释：  
router-link是类似超链接一样的东西  
router-view是组件将会被渲染到的地方

src/main.js  
它import了src/router，并且在Vue对象实例化的时候传入，随便贴个大概代码

```js
import Vue from "vue";
import App from "./App.vue";
import router from "./router";

Vue.config.productionTip = false;

new Vue({
  router,
  render: h => h(App)
}).$mount("#app");
```

## 动态路由匹配

很多时候，不同的URL可能会对应同一个页面，比如不同ID的用户页面都会使用同一个组件渲染  
因此可以使用动态路径参数来解决这个问题

例如，router的配置文件可以这样写

```js
import Vue from "vue";
import VueRouter from "vue-router";
import Home from "../views/Home.vue";
import User from "../views/User.vue";

Vue.use(VueRouter);

const routes = [
  {
    path: "/",
    name: "home",
    component: Home
  },
  {
    path: "/user/:id",
    name: "user",
    component: User
  }
];

const router = new VueRouter({
  mode: "history",
  base: process.env.BASE_URL,
  routes
});

export default router;
```

然后User.vue长这样

```html
<template>
  <div class="user">
    <h1>User ID: {{ $route.params.id }}</h1>
  </div>
</template>
```

这个东西使用path-to-regexp作为路径匹配引擎，支持正则匹配，这是它的GitHub链接  
[https://github.com/pillarjs/path-to-regexp](https://github.com/pillarjs/path-to-regexp)

然后路由匹配是按照定义的顺序决定的，先定义的先匹配  
因此可以在路由的最后配置404页面

## 嵌套路由

就是在子页面中使用路由

我们在路由配置中为user添加profile子路由，具体如下

```js
import Vue from "vue";
import VueRouter from "vue-router";
import Home from "../views/Home.vue";
import User from "../views/User.vue";
import Profile from "../views/Profile.vue"

Vue.use(VueRouter);

const routes = [
  {
    path: "/",
    name: "home",
    component: Home
  },
  {
    path: "/user/:id",
    name: "user",
    component: User,
    children: [
      {
        path: "profile",
        component: Profile
      }
    ]
  }
];

const router = new VueRouter({
  mode: "history",
  base: process.env.BASE_URL,
  routes
});
export default router;
```

同时修改User.vue，为它添加router-view

```html
<template>
  <div class="user">
    <h1>User ID: {{ $route.params.id }}</h1>
    <router-view />
  </div>
</template>
```

## 命名视图

有时候同一个页面中会有多个router-view，我们希望在不同的视图中显示不同的东西

路由配置如下

```js
import Vue from "vue";
import VueRouter from "vue-router";
import Home from "../views/Home.vue";
import User from "../views/User.vue";
import Profile from "../views/Profile.vue"
import Profile2 from "../views/Profile2.vue"

Vue.use(VueRouter);

const routes = [
  {
    path: "/",
    name: "home",
    component: Home
  },
  {
    path: "/user/:id",
    name: "user",
    component: User,
    children: [
      {
        path: "profile",
        components: {
          default: Profile,
          profile2: Profile2
        }
      }
    ]
  }
];

const router = new VueRouter({
  mode: "history",
  base: process.env.BASE_URL,
  routes
});
export default router;
```

User.vue如下

```html
<template>
  <div class="user">
    <h1>User ID: {{ $route.params.id }}</h1>
    <router-view />
    <router-view name="profile2"/>
  </div>
</template>
```

## 路由懒加载

对于一个路由中的组件，可以在被访问的时候再加载  
把import改成函数就行，比如下面这个

```js
import Vue from "vue";
import VueRouter from "vue-router";
import Home from "../views/Home.vue";
const User = () => import("../views/User.vue");
const Profile = () => import("../views/Profile.vue");
const Profile2 = () => import("../views/Profile2.vue");

Vue.use(VueRouter);

const routes = [
  {
    path: "/",
    name: "home",
    component: Home
  },
  {
    path: "/user/:id",
    name: "user",
    component: User,
    children: [
      {
        path: "profile",
        components: {
          default: Profile,
          profile2: Profile2
        }
      }
    ]
  }
];

const router = new VueRouter({
  mode: "history",
  base: process.env.BASE_URL,
  routes
});
export default router;
```

还可以把多个组件打包到一个chunk中，就能以chunk为单位异步加载，参考这个  
[https://webpack.js.org/guides/code-splitting/](https://webpack.js.org/guides/code-splitting/)
