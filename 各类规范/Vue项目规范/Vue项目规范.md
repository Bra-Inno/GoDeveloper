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

一个最常见的例子就是 **token**，我们需要在多个组件中使用 token，如果使用 props 传递的话，会显得非常繁琐，而使用状态管理则可以轻松实现。

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
- components 用于存放全局公共组件，**复用性极高**，功能单一并且相对独立，不涉及路由。
- views 用于存放页面级组件，功能较复杂，涉及业务逻辑，与路由绑定。
- 组件粒度上 components 偏小，views 偏大，可能包括多个组件。

### 3.7 结构总结
以上为从代码结构的角度对 Vue 项目进行了一个简单规范，而从功能角度来说：
- 如果要发送一个请求，那么要首先将请求的基本地址放到环境变量中，而请求实例在 api/request下，具体请求模块在 api/modules 下，最后在 views 中调用。
- 如果要管理状态，那么要首先在 store 中定义状态，然后在 views 中调用。
- 如果要管理路由，那么要首先在 router 中定义路由，然后在 views 中调用。
- 至于样式部分，全局样式放在 assets 中，局部样式直接写在组件内。

## 4 风格规范

除了项目结构规范之外，代码风格规范也至关重要。

以下部分主要来源于 Vue2 官方文档，将其修改的更符合 Vue3 使用。此部分依旧采用官方文档的分类方式，以规避错误、增强可读性、将选择认知成本最小化和有潜在危险的模式四类场景进行划分。

### 4.1 必要的（规避错误）
- 组件名为多个单词
组件名应始终是多个单词，避免与 HTML 元素冲突。

```javascript
// bad
<script>
import { defineComponent, ref } from 'vue'

export default defineComponent({
  name: 'Todo',
  setup() {
    const count = ref(0)
    return { count }
  }
})
</script>

// good
<script>
import { defineComponent, ref } from 'vue'

export default defineComponent({
  name: 'TodoItem',
  setup() {
    const count = ref(0)
    return { count }
  }
})
</script>


```

- Prop 定义尽量详细
Prop 定义应尽量详细，至少需要指定其类型。

```javascript
// bad
export default {
  props: {
    status: String
  }
}

// good
<script setup>
defineProps({
  status: {
    type: String,
    required: true
  }
})
</script>


```

- 为 v-for 设置键值
在组件上总是必须用 key 配合 v-for，以便维护内部组件及其子树的状态。甚至在元素上维护可预测的行为，比如动画中的对象固化 (object constancy)，也是一种好的做法。

```javascript
// bad
  <li v-for="todo in todos">
    {{ todo.text }}
  </li>

// good
  <li
    v-for="todo in todos"
    :key="todo.id"
  >
    {{ todo.text }}
  </li>

```
- 避免 v-if 和 v-for 用在一起
当 Vue 处理指令时，v-for 的优先级比 v-if 更高，所以当两者结合使用时，v-for 比 v-if 具有更高的优先级。这意味着 v-if 将分别重复运行于每个 v-for 循环中。如果我们只想为某些项渲染元素，我们应该在外面使用 computed 属性过滤列表。

```javascript
// bad
  <li
    v-for="user in users"
    v-if="user.isActive"
    :key="user.id"
  >
    {{ user.name }}
  </li>

  <li
    v-for="user in users"
    v-if="shouldShowUsers"
    :key="user.id"
  >
    {{ user.name }}
  </li>

//good
  <li
    v-for="user in activeUsers"
    :key="user.id"
  >
    {{ user.name }}
  </li>

    <li
    v-for="user in users"
    :key="user.id"
  >
    {{ user.name }}
  </li>

```

- 为组件样式设置作用域

如果你和其他开发者一起开发一个大型工程，或有时引入三方 HTML/CSS (比如来自 Auth0)，设置一致的作用域会确保你的样式只会运用在它们想要作用的组件上。

不止要使用 scoped attribute，使用唯一的 class 名可以帮你确保那些三方库的 CSS 不会运用在你自己的 HTML 上。比如许多工程都使用了 button、btn 或 icon class 名，所以即便你不使用类似 BEM 的策略，添加一个 app 专属或组件专属的前缀 (比如 ButtonClose-icon) 也可以提供很多保护。

```javascript
// bad
<template>
  <button class="btn btn-close">X</button>
</template>

<style>
.btn-close {
  background-color: red;
}
</style>

// good
<template>
  <button class="button button-close">X</button>
</template>

<!-- 使用 `scoped` attribute -->
<style scoped>
.button {
  border: none;
  border-radius: 2px;
}

.button-close {
  background-color: red;
}
</style>

<template>
  <button :class="[$style.button, $style.buttonClose]">X</button>
</template>

<!-- 使用 CSS Modules -->
<style module>
.button {
  border: none;
  border-radius: 2px;
}

.buttonClose {
  background-color: red;
}
</style>


<template>
  <button class="c-Button c-Button--close">X</button>
</template>

<!-- 使用 BEM 约定 -->
<style>
.c-Button {
  border: none;
  border-radius: 2px;
}

.c-Button--close {
  background-color: red;
}
</style>

```
那么我们采用的是 scoped attribute，这样可以确保我们的样式只会运用在它们想要作用的组件上。

- 私有属性定义
使用闭包变量实现私有属性，而不是通过命名约定。

```javascript
// bad
<script setup>
const $_privateMethod = () => {
  // ...
}
</script>

// good
<script setup>
import { usePrivateFeature } from './features'

// 私有逻辑封装在单独的组合式函数中
const { privateMethod } = usePrivateFeature()
</script>

```
### 4.2 强烈推荐（增强可读性）
- 组件文件
只要有能够拼接文件的构建系统，就把每个组件单独分成文件。

当你需要编辑一个组件或查阅一个组件的用法时，可以更快速的找到它。

```javascript
// bad
Vue.component('TodoList', {
  // ...
})

Vue.component('TodoItem', {
  // ...
})
```
```bash
# good
components/
|- TodoList.js
|- TodoItem.js

components/
|- TodoList.vue
|- TodoItem.vue

```

- 单文件组件名的大小写
单文件组件的文件名应该要么始终是单词大写开头 (PascalCase)，要么始终是横线连接 (kebab-case)。
那么我们采用大写开头的方式。

```bash
# bad
components/
|- mycomponent.vue

components/
|- myComponent.vue

# good
components/
|- MyComponent.vue

```
- 基础组件名
应用特定样式和约定的基础组件 (也就是展示类的、无逻辑的或无状态的组件) 应该全部以一个特定的前缀开头，比如 Base、App 或 V。

```bash
# bad
components/
|- MyButton.vue
|- VueTable.vue
|- Icon.vue

# good
components/
|- BaseButton.vue
|- BaseTable.vue
|- BaseIcon.vue

components/
|- AppButton.vue
|- AppTable.vue
|- AppIcon.vue

components/
|- VButton.vue
|- VTable.vue
|- VIcon.vue

```
- 单例组件名
只应该拥有单个活跃实例的组件应该以 The 前缀命名，以示其唯一性。

```bash
# bad
components/
|- Heading.vue
|- MySidebar.vue

# good

components/
|- TheHeading.vue
|- TheSidebar.vue

```

- 紧密耦合的组件名
和父组件紧密耦合的子组件应该以父组件名作为前缀命名。
如果一个组件只在某个父组件的场景下有意义，这层关系应该体现在其名字上。因为编辑器通常会按字母顺序组织文件，所以这样做可以把相关联的文件排在一起。

```bash
# bad
components/
|- TodoList.vue
|- TodoItem.vue
|- TodoButton.vue

components/
|- SearchSidebar.vue
|- NavigationForSearchSidebar.vue

# good
components/
|- TodoList.vue
|- TodoListItem.vue
|- TodoListItemButton.vue

components/
|- SearchSidebar.vue
|- SearchSidebarNavigation.vue

```

- 组件名中的单词顺序
组件名应该以高级别的 (通常是一般化描述的) 单词开头，以描述性的修饰词结尾。

```bash
# bad
components/
|- ClearSearchButton.vue
|- ExcludeFromSearchInput.vue
|- LaunchOnStartupCheckbox.vue
|- RunSearchButton.vue
|- SearchInput.vue
|- TermsCheckbox.vue

# good
components/
|- SearchButtonClear.vue
|- SearchButtonRun.vue
|- SearchInputQuery.vue
|- SearchInputExcludeGlob.vue
|- SettingsCheckboxTerms.vue
|- SettingsCheckboxLaunchOnStartup.vue

```

- 自闭合组件
在单文件组件、字符串模板和 JSX 中，没有内容的组件应该是自闭合的——但在 DOM 模板里永远不要这样做。

```javascript
// bad
<!-- 在单文件组件、字符串模板和 JSX 中 -->
<MyComponent></MyComponent>

<!-- 在 DOM 模板中 -->
<my-component/>

// good
<!-- 在单文件组件、字符串模板和 JSX 中 -->
<MyComponent/>

<!-- 在 DOM 模板中 -->
<my-component></my-component>

```

- 模板中的组件名大小写
对于绝大多数项目来说，在单文件组件和字符串模板中组件名应该总是 PascalCase 的——但是在 DOM 模板中总是 kebab-case 的。
那么我们使用以下方式作为规范：

```javascript
// bad
<!-- 在单文件组件和字符串模板中 -->
<mycomponent/>

<!-- 在单文件组件和字符串模板中 -->
<myComponent/>

<!-- 在 DOM 模板中 -->
<MyComponent></MyComponent>

// good
<!-- 在所有地方 -->
<my-component></my-component>

```

- JS/JSX 中的组件名大小写
在 JavaScript 和 JSX 中始终使用 PascalCase 来命名组件。

```javascript
// good
import MyComponent from './MyComponent.vue'

export default {
  name: 'MyComponent',
  // ...
}

app.component('MyComponent', {
  // ...
})
```

- 完整单词的组件名
组件名应该倾向于完整单词而不是缩写。

```bash
# bad
components/
|- SdSettings.vue
|- UProfOpts.vue

# good
components/
|- StudentDashboardSettings.vue
|- UserProfileOptions.vue

```

- Prop 名大小写
在声明 prop 的时候，其命名应该始终使用 camelCase，而在模板和 JSX 中应该始终使用 kebab-case。

```javascript
// bad
props: {
  'greeting-text': String
}

// good
props: {
  greetingText: String
}

```

- 多个 attribute 的元素
多个 attribute 的元素应该分多行写，每个 attribute 一行。

```javascript
// bad
<img src="https://vuejs.org/images/logo.png" alt="Vue Logo">
<MyComponent foo="a" bar="b" baz="c"/>

// good
<img
  src="https://vuejs.org/images/logo.png"
  alt="Vue Logo"
>
<MyComponent
  foo="a"
  bar="b"
  baz="c"
/>

```

- 简单的计算属性
简单的计算属性应该始终使用 getter/setter 函数，这样可以确保在模板中保持简洁。

```javascript
// bad
// 选项式 API
export default {
  computed: {
    basePrice() {
      return this.manufactureCost / (1 - this.profitMargin)
    },
    discount() {
      return this.basePrice * (this.discountPercent || 0)
    },
    finalPrice() {
      return this.basePrice - this.discount
    }
  }
}

// 组合式 API
import { computed } from 'vue'

export default {
  setup() {
    const basePrice = computed(() => 
      manufactureCost.value / (1 - profitMargin.value)
    )

    const discount = computed(() => 
      basePrice.value * (discountPercent.value || 0)
    )

    const finalPrice = computed(() => 
      basePrice.value - discount.value
    )

    return {
      basePrice,
      discount,
      finalPrice
    }
  }
}

// good
// 选项式 API
export default {
  computed: {
    finalPrice: {
      get() {
        return this.basePrice - this.discount
      },
      set(newValue) {
        // 处理设置逻辑
      }
    }
  }
}

// 组合式 API
const finalPrice = computed({
  get: () => basePrice.value - discount.value,
  set: (newValue) => {
    // 处理设置逻辑
  }
})

```

- 带引号的 attribute 值
非空 HTML attribute 值应该始终带引号 (单引号或双引号，以 JS 中未使用的为准)。
在 HTML 中不带空格的 attribute 值是可以没有引号的，但这鼓励了大家在特征值里不写空格，导致可读性变差。

```javascript
// bad
<input type=text>

<AppSidebar :style={width:sidebarWidth+'px'}>

// good
<input type="text">

<AppSidebar :style="{ width: sidebarWidth + 'px' }">

```

- 指令缩写
指令缩写 (用 : 表示 v-bind:、用 @ 表示 v-on: 和用 # 表示 v-slot:) 应该要么都用要么都不用。
那么我们采用全都进行缩写作为规范：

```javascript
// bad
<input
  v-bind:value="newTodoText"
  :placeholder="newTodoInstructions"
>

// good
<input
  :value="newTodoText"
  :placeholder="newTodoInstructions"
>

```

### 4.3 推荐（将选择和认知成本最小化）

- 组件/实例的选项顺序推荐
组件的选项顺序应保持一致，以下是推荐的顺序：

1. 副作用（触发组件外的影响）
   - `el`
  
2. 全局感知（要求组件以外的知识）
   - `name`
   - `parent`

3. 模板修改器（改变模板的编译方式）
   - `delimiters`
   - `comments`

4. 模板依赖（模板内使用的资源）
   - `components`
   - `directives`
   - `filters`

5. 组合（向选项里合并 property）
   - `extends`
   - `mixins`

6. 接口（组件的接口）
   - `inheritAttrs`
   - `model`
   - `props`

7. 本地状态（本地的响应式 property）
   - `data`
   - `computed`

8. 事件（通过响应式事件触发的回调）
   - `watch`

9. 生命周期钩子（按照它们被调用的顺序）
    - `beforeCreate`
    - `created`
    - `beforeMount`
    - `mounted`
    - `beforeUpdate`
    - `updated`
    - `activated`
    - `deactivated`
    - `beforeDestroy`
    - `destroyed`（`beforeDestroy` 和 `destroyed` 在 Vue 3 中被 `beforeUnmount` 和 `unmounted` 替代）

10. 非响应式的 property（不依赖响应系统的实例 property）
    - `methods`

11. 渲染（组件输出的声明式描述）
    - `template/render`
    - `renderError`

- 元素属性的顺序推荐
元素（包括组件）的属性应有统一的顺序，以下是推荐的顺序：

1. 定义（提供组件的选项）
   - `is`

2. 列表渲染（创建多个变化的相同元素）
   - `v-for`

3. 条件渲染（元素是否渲染/显示）
   - `v-if`
   - `v-else-if`
   - `v-else`
   - `v-show`
   - `v-cloak`

4. 渲染方式（改变元素的渲染方式）
   - `v-pre`
   - `v-once`

5. 全局感知（需要超越组件的知识）
   - `id`

6. 唯一的 attribute（需要唯一值的 attribute）
   - `ref`
   - `key`

7. 双向绑定（把绑定和事件结合起来）
   - `v-model`

8. 其它 attribute（所有普通的绑定或未绑定的 attribute）

9. 事件（组件事件监听器）
   - `v-on`

10. 内容（覆写元素的内容）
    - `v-html`
    - `v-text`

- 组件/实例选项中的空行推荐
在多个 property 之间增加空行，以提高可读性和导航性。尤其是在选项较多时，适当的空行可以使代码更易于理解。

- 单文件组件的顶级元素顺序推荐
单文件组件的 `<script>`、`<template>` 和 `<style>` 标签顺序应保持一致，且 `<style>` 标签应放在最后。

**好例子**
```vue
<script>
// ...
</script>
<template>
  <!-- ... -->
</template>
<style scoped>
/* ... */
</style>
```


### 4.4 谨慎使用（有潜在危险的模式）
- 没有在 `v-if/v-else-if/v-else` 中使用 `key`
当一组 `v-if` + `v-else` 的元素类型相同时，最好使用 `key`。

- scoped 中的元素选择器谨慎使用
在 scoped 样式中，类选择器比元素选择器更好。

- 隐性的父子组件通信谨慎使用
优先通过 prop 和事件进行父子组件之间的通信。

- 非 Flux 的全局状态管理谨慎使用
应优先通过 Vuex 管理全局状态，而不是通过 `this.$root` 或全局事件总线。