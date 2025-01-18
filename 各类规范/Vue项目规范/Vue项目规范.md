[TOC]

# Vue 项目规范

## 1 简介

在使用 Vue 进行前端开发时，为了回避错误、小纠结和反模式，以及为了能够更好的多人协同，笔者参考了一些优秀开源项目，以及 Vue2 风格指南，整理了此份 Vue 项目规范，设计项目结构以及代码风格。

这份规范我自认为不会是最理想的，仅仅是根据过往经验以及个人取舍总结而成，为了让我们团队内部多人协同开发更加顺畅。 

如果有遗漏或错误欢迎指正。

## 2 参考项目及文章

以下为主要参考文章及项目：

- [Vue官方文档](https://v2.cn.vuejs.org/v2/style-guide/)
- [Vue3后台管理系统](https://github.com/lin-xin/vue-manage-system)
- [vue2网易云](https://github.com/javaswing/NeteaseCloudWebApp.git)
- [vue2饿了么](https://github.com/bailicangdu/vue2-elm)

##  3 项目整体结构

在使用 Vue 进行开发时 src 目录为项目主要目录，故只选取此部分。

以下大致为 src 目录的基本结构，我将按照文件目录的结构对每一部分进行详细说明。

```bash
src/
├── api/               # API 接口管理
│
├── assets/            # 静态资源
│
├── components/        # 全局公共组件
│
├── router/            # 路由配置
│   └── index.js       # 路由主文件
│
├── store/             # 状态管理
│
├── utils/             # 工具函数
│
├── views/             # 页面视图
│
├── App.vue            # 根组件
└── main.js            # 入口文件
```

另外，像 VUE_APP_BASE_API 等环境变量配置，最好放在项目根目录下统一管理。

```bash
项目根目录/
├── .env                  # 所有环境都会加载
├── .env.development      # 开发环境
├── .env.production       # 生产环境
└── .env.staging          # 测试环境
```
按照环境的不同，我们可以在不同文件中配置不同的环境变量，然后在项目中通过 **process.env.VUE_APP_BASE_API** 进行获取。

### 3.1 api

此部分用于集中管理所有 API 接口，为了高效与便捷我们选择使用 axios 统一处理各类网络请求，并根据功能的不同按模块划分不同的 API 文件。

- 为什么不使用 fetch 与 ajax？

    fetch 虽然是官方原生 API ，但是 axios 与其相比具有以下特性：
    - axios 能够自动转换 JSON 数据，而 fetch 需要手动调用 json() 方法。
    - axios 支持请求取消、超时、拦截器、错误处理等功能，而 fetch 不支持。
    - axios 能够自动处理错误状态码，例如 404、500 等，而 fetch 只有在网络错误时才会 reject。

    ajax 是 jQuery 的 API，使用起来不够灵活，而且 jQuery 体积较大，不适合用于 Vue 项目。

```bash
api/                         # API 接口目录
├── request/                 # 请求配置
│   ├── index.js             # axios 实例和拦截器配置
│   └── cancel.js            # 请求取消处理
├── modules/                 # API 模块
│   ├── user.js              # 用户相关接口
│   ├── product.js           # 商品相关接口
│   └── order.js             # 订单相关接口
└── index.js                 # API 统一导出
```

#### 3.1.1 request

此部分用于配置 axios 实例和拦截器。
以下为代码示例：

```javascript
// api/request/index.js
import axios from 'axios'
import { Message } from 'element-ui' // 假设使用element-ui

const request = axios.create({
  baseURL: process.env.VUE_APP_BASE_API,
  timeout: 600000,
  headers: {
    'Accept': 'application/json'
  }
})

// 请求拦截器
request.interceptors.request.use(
  config => {
    const token = localStorage.getItem('token')
    if (token) {
      config.headers['Authorization'] = `Bearer ${token}`  // 规范化token格式
    }
    return config
  },
  error => {
    return Promise.reject(error)
  }
)

// 响应拦截器
request.interceptors.response.use(
  response => {
    const res = response.data
    if (res.code !== 200) {
      Message.error(res.message || '错误')
      return Promise.reject(new Error(res.message || '错误'))
    }
    return res
  },
  error => {
    Message.error(error.message || '网络错误')
    return Promise.reject(error)
  }
)

export default request

// api/request/cancel.js
import axios from 'axios'

export const addCancelToken = config => {
  const source = axios.CancelToken.source()
  config.cancelToken = source.token
  return source
}

export const cancelRequest = (source, msg = '请求已取消') => {
  source.cancel(msg)
}

```

#### 3.1.2 modules

在一个项目中我们往往需要调用多个不同的 API 接口，我们可以根据功能的不同进行划分，例如用户相关接口就全都放在 user.js 文件中，商品相关接口放在 product.js 文件中。

假设我们有一个产品接口，我们可以在 product.js 文件中这样写：

```javascript
// api/modules/product.js
import request from '../request'
import { addCancelToken } from '../request/cancel'

export const productApi = {
  getProductList: (params) => {
    const config = { params }
    const source = addCancelToken(config)
    return {
      request: request.get('/product/list', config),
      source
    }
  },
  getProductDetail: (id) => {
    const config = {}
    const source = addCancelToken(config)
    return {
      request: request.get(`/product/${id}`, config),
      source
    }
  }
}
```

#### 3.1.3 index.js

api 目录下的 index.js 文件用于统一导出所有 API 模块，方便在其他地方引用。

```javascript
// api/index.js
import request from './request'
import { userApi } from './modules/user'
import { productApi } from './modules/product'
import { orderApi } from './modules/order'

export {
  request,
  userApi,
  productApi,
  orderApi
}
```

### 3.2 assets

此部分用于存放各类静态资源，例如图片、字体、样式等。但是其中的 css 代码只存放全局样式，局部样式直接放在组件内。

### 3.3 store

此部分用于存放 Vuex 或 Pinia 这样的状态管理相关的代码，由于习惯我们使用 **Pinia**。
- 什么是状态？
    状态是指应用程序中的数据，例如用户信息、商品信息、购物车信息等。

- 为什么要使用状态管理？
    在 Vue 中，我们可以通过 props 和 emit 来实现父子组件之间的通信，但是当组件之间的关系变得复杂时，这种方式就显得不够灵活，状态管理可以帮助我们更好的管理组件之间的数据。

一个最常见的例子就是 token，我们需要在多个组件中使用 token，如果使用 props 传递的话，会显得非常繁琐，而使用状态管理则可以轻松实现。

```javascript
// store/auth.js
import { defineStore } from 'pinia'

export const useAuthStore = defineStore('auth', {
  state: () => ({
    token: localStorage.getItem('token') || '',
    userInfo: {}
  }),
  
  actions: {
    // 设置token
    setToken(token) {
      this.token = token
      localStorage.setItem('token', token)
    },
    
    // 清除token
    clearToken() {
      this.token = ''
      localStorage.removeItem('token')
    },
    
    // 登录
    async login(credentials) {
      try {
        const response = await api.login(credentials)
        this.setToken(response.data.token)
        this.userInfo = response.data.user
      } catch (error) {
        throw error
      }
    },
    
    // 登出
    logout() {
      this.clearToken()
      this.userInfo = {}
    }
  },
  
  getters: {
    isAuthenticated: (state) => !!state.token
  }
})
```

### 3.4 utils

此处用于存放项目中可复用的工具函数和辅助方法，例如密码正则、日期格式化、防抖节流等。

```javascript
// utils/password.js
export const passwordReg = /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)[a-zA-Z\d]{8,}$/

// utils/date.js
export const formatDate = (date) => {
  const d = new Date(date)
  return `${d.getFullYear()}-${d.getMonth() + 1}-${d.getDate()}`
}

```

### 3.5 router

此部分主要用于管理路由设置，负责各页面之间的跳转。

```javascript
// router/index.js
import { createRouter, createWebHistory } from 'vue-router'

// 引入布局组件
import Layout from '@/layout/index.vue'

// 路由懒加载方式引入组件
const Login = () => import('@/views/login/index.vue')
const Home = () => import('@/views/home/index.vue')
const NotFound = () => import('@/views/error/404.vue')

// 基础路由
export const constantRoutes = [
  {
    path: '/login',
    name: 'Login',
    component: Login,
    meta: {
      title: '登录',
      hidden: true // 是否在侧边栏隐藏
    }
  },
  {
    path: '/',
    component: Layout,
    redirect: '/home',
    children: [
      {
        path: 'home',
        name: 'Home',
        component: Home,
        meta: {
          title: '首页',
          icon: 'home',
          affix: true // 是否固定在 tabs
        }
      }
    ]
  },
  // 用户管理模块
  {
    path: '/user',
    component: Layout,
    redirect: '/user/list',
    meta: {
      title: '用户管理',
      icon: 'user',
      roles: ['admin'] // 可访问该路由的角色
    },
    children: [
      {
        path: 'list',
        name: 'UserList',
        component: () => import('@/views/user/list.vue'),
        meta: {
          title: '用户列表',
          keepAlive: true // 是否缓存该路由
        }
      },
      {
        path: 'add',
        name: 'UserAdd',
        component: () => import('@/views/user/add.vue'),
        meta: {
          title: '添加用户'
        }
      }
    ]
  },
  // 系统设置模块
  {
    path: '/system',
    component: Layout,
    redirect: '/system/menu',
    meta: {
      title: '系统设置',
      icon: 'setting'
    },
    children: [
      {
        path: 'menu',
        name: 'Menu',
        component: () => import('@/views/system/menu.vue'),
        meta: {
          title: '菜单管理'
        }
      },
      {
        path: 'role',
        name: 'Role',
        component: () => import('@/views/system/role.vue'),
        meta: {
          title: '角色管理'
        }
      }
    ]
  },
  // 404 页面
  {
    path: '/:pathMatch(.*)*',
    name: 'NotFound',
    component: NotFound,
    meta: {
      title: '404',
      hidden: true
    }
  }
]

// 创建路由实例
const route
```

### 3.6 components与views

在最开始我接触 Vue 时往往区分不出来 components 和 views 的区别，往往会将所有组件要么全都放在 components 中，要么全都放在 views 中，后来我总结了一下，components 是用于存放全局公共组件，例如按钮、输入框、弹窗等，而 views 是用于存放页面视图，例如首页、用户管理、商品管理等。这么说可能还是比较笼统，那么我来举个例子，现在有：

- Button.vue
- Input.vue
- Header.vue
- Footer.vue
- index.vue
- Login.vue
- Chat.vue

那么我们不难发现 Button、Input、Header、Footer 这四个组件基本在各个页面中都会出现，复用性强，且一般不涉及业务逻辑和路由，所以这四个组件就应该放在 components 中，而 index、Login、Chat 这三个组件基本上是独立的也，面，也各有各的功能以及路由，所以这三个组件就应该放在 views 中。

那么我们总结一下：
	components 与 views 的**相同之处**在于：

- 都是用于存放 Vue 组件的目录
- 都有自己的样式代码以及基本交互逻辑

​	而**不同之处**在于：
- components 用于存放全局公共组件，复用性极高，功能单一并且相对独立，不涉及路由。
- views 用于存放页面级组件，功能较复杂，涉及业务逻辑，与路由绑定。
- 组件粒度上 components 偏小，views 偏大，可能包括多个组件。

### 3.7 结构总结
以上为从代码结构的角度对 Vue 项目进行了一个简单规范，而从功能角度来说：
- 如果要发送一个请求，那么要首先将请求的基本地址放到环境变量中，而请求实例在 api/request下，具体请求模块在 api/modules 下，最后在 views 中调用。
- 如果要管理状态，那么要首先在 store 中定义状态，然后在 views 中调用。
- 如果要管理路由，那么要首先在 router 中定义路由，然后在 views 中调用。
- 至于样式部分，全局样式放在 assets 中，局部样式直接写在组件内。

## 4 风格规范