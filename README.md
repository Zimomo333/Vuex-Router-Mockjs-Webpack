# Vuex-Router-Webpack 模块化开发

### 项目目的

不使用vue-cli脚手架，从头搭建模块化开发环境



### 项目功能

1. 导航栏（折叠）
2. 面包屑
3. Vue-Router（路由、拦截功能）
4. Vuex（中心化登录权限）
5. token存入Cookie，用户信息存入localStorage
6. Wepack 代码压缩，热加载本地服务器，以Hash值命名包更新缓存，优化分包策略



### 项目依赖

1. Vue
2. Vue Router
3. Vuex
4. axios异步请求、js-cookie操作Cookies
5. Element-UI
6. webpack

```shell
npm init
npm i vue -S

npm i element-ui -S

npm i vue-router -S

npm i vuex -S

npm i axios -S
npm i js-cookie -S

npm i webpack webpack-dev-server -D         // 热加载本地server，用于运行打包后的dist资源

<!-- webpack loader -->
npm i vue-loader vue-template-compiler -D   // 解析vue文件、模板
npm i css-loader style-loader -D            // 解析Element-UI的CSS文件
npm i file-loader -D                        // 解析Element-UI的字体文件

<!-- webpack plugin -->
npm i html-webpack-plugin -D                // 自动生成注入js的index.html
```



### 目录结构

<div><img src="https://raw.githubusercontent.com/Zimomo333/Vuex-Router-Webpack/master/picture/directory.PNG"></div>

结构说明：

| 文件/目录名         | 作用                                        |
| ------------------- | ------------------------------------------- |
| `webpack.config.js` | `webpack` 配置文件                          |
| `routes.js`         | `Vue Router` 配置文件                       |
| `store.js`          | `Vuex` 配置文件                             |
| `main.js`           | 全局配置，`webpack`打包入口                 |
| `request.js`           | 二次封装axios，错误码处理，header设置token |
| `auth.js`           | Cookies、localStorage 操作 |
| `App.vue`           | `Vue` 根组件                                |
| `public/index.html` | `HtmlWebpackPlugin` 自定义`index.html` 模板 |
| `views`目录         | 页面组件（业务页面）                        |
| `components`目录    | 公用组件（导航栏、面包屑）                  |
| `api`目录  | 请求接口目录                                |
| `dist`目录          | `webpack`打包输出目录                       |
| `node_modules`目录  | `npm` 模块下载目录                          |







## Vue-Router

### router.js

```javascript
import Vue from 'vue'
import VueRouter from 'vue-router'
import store from './store'
import { getToken } from './utils/auth'

Vue.use(VueRouter)

export const routes = [
  {
    path: '//',     // 转义 / ，防止自动省略为空
    component: () => import('./views/home.vue'),
    meta: { 
      title: '首页',
      icon: 'el-icon-s-order'
    },
    hidden: true,
    children: [
      { 
        path: '/info',
        component: () => import('./views/info.vue'),
        meta: { 
            title: '个人中心',
            icon: 'el-icon-user-solid'
        }
      },
      {
        path: '/orders',        // 以 / 开头的嵌套路径会被当作根路径
        component: () => import('./views/orders/index.vue'),    // 可写成{render: (e) => e("router-view")}，避免新建空router-view文件
        meta: { 
            title: '订单管理',
            icon: 'el-icon-s-order'
        },
        children: [
          {
            path: 'my-orders',      // 子路由不要加 /
            component: () => import('./views/orders/myOrders.vue'),
            meta: { 
                title: '我的订单',
                icon: 'el-icon-s-order'
            }
          },
          {
            path: 'submit',
            component: () => import('./views/orders/submit.vue'),
            meta: { 
                title: '提交订单',
                icon: 'el-icon-s-order'
            }
          }
        ]
      },
    ]
  },
  {
    path: '/login',
    component: () => import('./views/login.vue'),
    hidden: true
  }
]

const router = new VueRouter({
    routes // (缩写) 相当于 routes: routes
})

//路由全局前置守卫，验证token
router.beforeEach(async(to, from, next) => {
  
    // determine whether the user has logged in
    const hasToken = getToken()
  
    if (hasToken) {
      if (to.path === '/login') {
        // if is logged in, redirect to the home page
        next('/')
      } else {
        const hasGetUserInfo = store.getters.name
        if (hasGetUserInfo) {
          next()
        } else {
          try {
            // get user info
            await store.dispatch('getInfo')
  
            next()
          } catch (error) {
            await store.dispatch('resetToken')
            next('/login')
          }
        }
      }
    } else {
        /* has no token*/
        if (to.path === '/login') {   //next()才能跳出循环
            next()
        } else {
            next('/login')
        }
    }
})

export default router
```







## Vuex

### store.js

```js
import Vue from 'vue'
import Vuex from 'vuex'
import { login, logout, getInfo } from './api/user'
import { getToken, setToken, removeToken, setUserInfo, removeUserInfo } from './utils/auth'

Vue.use(Vuex)

const store = new Vuex.Store({
    state: {
        token: getToken(),
        name: '',
        avatar: ''
    },
    getters: {
        token: state => state.token,
        avatar: state => state.avatar,
        name: state => state.name
    },
    mutations: {
        RESET_STATE: (state) => {
            state.token = ''
            state.name = ''
            state.avatar = ''
        },
        SET_TOKEN: (state, token) => {
            state.token = token
        },
        SET_USERINFO: (state, userInfo) => {
            state.name = userInfo.name
            state.avatar = userInfo.avatar
        },
    },
    actions: {
        login({ commit }, loginForm) {
            const { username, password } = loginForm
            return new Promise((resolve, reject) => {
                login({ username: username.trim(), password: password }).then(response => {
                    const { data } = response
                    commit('SET_TOKEN', data.token)
                    setToken(data.token)
                    resolve()
                }).catch(error => {
                    reject(error)
                })
            })
        },
        getInfo({ commit, state }) {
            return new Promise((resolve, reject) => {
                getInfo(state.token).then(response => {
                    const { data } = response

                    if (!data) {
                        reject('Verification failed, please Login again.')
                    }

                    const { userInfo } = data
                    commit('SET_USERINFO', userInfo)
                    setUserInfo(userInfo)
                    resolve(data)
                }).catch(error => {
                    reject(error)
                })
            })
        },
        logout({ commit, state }) {
            return new Promise((resolve, reject) => {
                logout(state.token).then(() => {
                    removeToken()
                    removeUserInfo()
                    commit('RESET_STATE')
                    resolve()
                }).catch(error => {
                    reject(error)
                })
            })
        },
        // remove token
        resetToken({ commit }) {
            return new Promise(resolve => {
                removeToken() 
                commit('RESET_STATE')
                resolve()
            })
        }
    }
})

export default store
```







## Webpack

### webpack.config.js

```javascript
const path = require('path')
const VueLoaderPlugin = require('vue-loader/lib/plugin')
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = {
    entry: './src/main.js',
    output: {
        path: path.resolve(__dirname, './dist'),
        filename: 'bundle.js'               // 打包输出的js包名
    },
    devServer: {
        contentBase: './dist'               // server运行的资源目录
    },
    module: {
        rules: [
            {
                test: /\.vue$/,
                loader: 'vue-loader'
            },
            {
                test: /\.css$/,             // 处理ElementUI样式
                loader: "style-loader!css-loader"
            },
            {
                test: /\.(woff|ttf)$/,	    // 处理ElementUI字体
                loader: 'file-loader'
            }
        ]
    },
    plugins: [
        new VueLoaderPlugin(),              // vue-loader伴生插件，必须有！！！
        new HtmlWebpackPlugin({             // 自动生成注入js的index主页
            title: 'Vuex-Router-Webpack demo',
            template: './public/index.html' // 自定义index模板
        })
    ]
}
```



### `public/index.html`模板

默认生成的`index.html `没有 id="app" 挂载点，必须使用自定义模板

```html
<!DOCTYPE html>
<html>
    <head>
        <meta charset="UTF-8">
        <title></title>
    </head>
    <body>
        <div id="app">		<!-- App.vue根组件 挂载点 -->
        </div>
    </body>
</html>
```



### 以Hash值命名包更新缓存

```javascript
{
  output: {
    /*
    代码中引用的文件（js、css、图片等）会根据配置合并为一个或多个包，我们称一个包为 chunk。
    每个 chunk 包含多个 modules。无论是否是 js，webpack 都将引入的文件视为一个 module。
    chunkFilename 用来配置这个 chunk 输出的文件名。

    [chunkhash]：这个 chunk 的 hash 值，文件发生变化时该值也会变。使用 [chunkhash] 作为文件名可以防止浏览器读取旧的缓存文件。

    还有一个占位符 [id]，编译时每个 chunk 会有一个id。
    我们在这里不使用它，因为这个 id 是个递增的数字，增加或减少一个chunk，都可能导致其他 chunk 的 id 发生改变，导致缓存失效。
    */
    chunkFilename: '[chunkhash].js',
  }
  
  plugins: [
    /*
    使用文件路径的 hash 作为 moduleId。
    虽然我们使用 [chunkhash] 作为 chunk 的输出名，但仍然不够。
    因为 chunk 内部的每个 module 都有一个 id，webpack 默认使用递增的数字作为 moduleId。
    如果引入了一个新文件或删掉一个文件，可能会导致其他文件的 moduleId 也发生改变，
    那么受影响的 module 所在的 chunk 的 [chunkhash] 就会发生改变，导致缓存失效。
    因此使用文件路径的 hash 作为 moduleId 来避免这个问题。
    */
    new webpack.HashedModuleIdsPlugin()
  ],

  optimization: {
    /*
    上面提到 chunkFilename 指定了 chunk 打包输出的名字，那么文件名存在哪里了呢？
    它就存在引用它的文件中。这意味着一个 chunk 文件名发生改变，会导致引用这个 chunk 文件也发生改变。

    runtimeChunk 设置为 true, webpack 就会把 chunk 文件名全部存到一个单独的 chunk 中，
    这样更新一个文件只会影响到它所在的 chunk 和 runtimeChunk，避免了引用这个 chunk 的文件也发生改变。
    */
    runtimeChunk: true,

    splitChunks: {
      /*
      默认 entry 的 chunk 不会被拆分
      因为我们使用了 html-webpack-plugin 来动态插入 <script> 标签，entry 被拆成多个 chunk 也能自动被插入到 html 中，
      所以我们可以配置成 all, 把 entry chunk 也拆分了
      */
      chunks: 'all'
    }
  }
}
```

### 测试

**第一次构建**

<img src="https://raw.githubusercontent.com/Zimomo333/Vuex-Router-Webpack/master/picture/first_hash.JPG" width="300px" height="540px" />

新增一个test.vue，路由文件新增一个路径，**第二次构建**

<img src="https://raw.githubusercontent.com/Zimomo333/Vuex-Router-Webpack/master/picture/second_hash.JPG" width="300px" height="580px" />

**可以看到大部分包的hash值不变，缓存仍有效**



### 优化分包策略（参考自 字节跳动 花裤衩 大佬）

按照体积大小、共用率、更新频率重新划分包，使其尽可能的利用浏览器缓存。

<img src="https://user-gold-cdn.xitu.io/2018/8/7/16513e5b6a73ac96?w=1482&h=756&f=jpeg&s=150509" width="700px" height="357px" />

- 基础类库 chunk-libs

它是构成我们项目必不可少的一些基础类库，比如 `vue`+`vue-router`+`vuex`+`axios` 这种标准的全家桶，它们的升级频率都不高，但每个页面都需要它们。（一些全局被共用的，体积不大的第三方库也可以放在其中：比如 nprogress、js-cookie、clipboard 等）

- UI 组件库

理论上 UI 组件库也可以放入 libs 中，但这里单独拿出来的原因是： 它实在是比较大，不管是 `Element-UI`还是`Ant Design` gizp 压缩完都可能要 200kb 左右，它可能比 libs 里面所有的库加起来还要大不少，而且 UI 组件库的更新频率也相对的比 libs 要更高一点。我们不时的会升级 UI 组件库来解决一些现有的 bugs 或使用它的一些新功能。所以建议将 UI 组件库也单独拆成一个包。

- 自定义组件/函数 chunk-commons

这里的 commons 主要分为 **必要**和**非必要**。

必要组件是指那些项目里必须加载它们才能正常运行的组件或者函数。比如你的路由表、全局 state、全局侧边栏/Header/Footer 等组件、自定义 Svg 图标等等。这些其实就是你在入口文件中依赖的东西，它们都会默认打包到`app.js`中。

非必要组件是指被大部分页面使用，但在入口文件 entry 中未被引入的模块。比如：一个管理后台，你封装了很多 select 或者  table 组件，由于它们的体积不会很大，它们都会被默认打包到到每一个懒加载页面的 chunk  中，这样会造成不少的浪费。你有十个页面引用了它，就会包重复打包十次。所以应该将那些被大量共用的组件单独打包成`chunk-commons`。

不过还是要结合具体情况来看。一般情况下，你也可以将那些*非必要组件函数*也在入口文件 entry 中引入，和*必要组件函数*一同打包到`app.js`之中也是没什么问题的。

- 低频组件

低频组件和上面的共用组件 `chunk-commons` 最大的区别是，它们只会在一些特定业务场景下使用，比如富文本编辑器、`js-xlsx`前端 excel 处理库等。一般这些库都是第三方的且大于 30kb，所以 webpack 4 会默认打包成一个独立的 bundle。也无需特别处理。小于 30kb 的情况下会被打包到具体使用它的页面 bundle 中。

- 业务代码

这部分就是我们平时经常写的业务代码。一般都是按照页面的划分来打包，比如在 vue 中，使用[路由懒加载](https://router.vuejs.org/zh/guide/advanced/lazy-loading.html)的方式加载页面 `component: () => import('./Foo.vue')` webpack 默认会将它打包成一个独立的 bundle。

配置代码：

```javascript
splitChunks: {
  chunks: "all",
  cacheGroups: {
    libs: {
      name: "chunk-libs",
      test: /[\\/]node_modules[\\/]/,
      priority: 10,
      chunks: "initial" // 只打包初始时依赖的第三方
    },
    elementUI: {
      name: "chunk-elementUI", // 单独将 elementUI 拆包
      priority: 20, // 权重要大于 libs 和 app 不然会被打包进 libs 或者 app
      test: /[\\/]node_modules[\\/]element-ui[\\/]/
    },
    commons: {
      name: "chunk-commons",
      test: path.resolve("src/components"), // 可自定义拓展你的规则
      minChunks: 2, // 最小共用次数
      priority: 5,
      reuseExistingChunk: true
    }
  }
};
```

### 测试

使用 **webpack-bundle-analyzer** 插件可视化分析包结构

```shell
npm install --save-dev webpack-bundle-analyzer
```

```javascript
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;
 
module.exports = {
  plugins: [
    new BundleAnalyzerPlugin()
  ]
}
```

**打包前**

<div align=center><img src="https://raw.githubusercontent.com/Zimomo333/Vuex-Router-Webpack/master/picture/before_split.PNG"></div>

**打包后**

<div align=center><img src="https://raw.githubusercontent.com/Zimomo333/Vuex-Router-Webpack/master/picture/after_split.PNG"></div>







## 组件

### SidebarItem.vue(导航栏组件)

组件自调用实现嵌套导航栏

传递basePath记录父路由路径，与子路由拼接成完整路径

```vue
<template>
  <!-- 隐藏不需要显示的路由 -->
  <div v-if="!item.hidden">
    <!-- 如果没有子路由 -->
    <template v-if="!item.children">
        <el-menu-item :key="item.path" :index="resolvePath(item.path)">
          <i :class="item.meta.icon"></i>
          <span slot="title">{{item.meta.title}}</span>
        </el-menu-item>
    </template>
    <!-- 如果有子路由，渲染子菜单 -->
    <el-submenu v-else :index="resolvePath(item.path)">
      <template slot="title">
          <i :class="item.meta.icon"></i>
          <span slot="title">{{item.meta.title}}</span>
      </template>
      <sidebar-item v-for="child in item.children" :key="child.path" :item="child" :basePath="resolvePath(item.path)" />
    </el-submenu>
  </div>
</template>

<script>
import path from 'path'

export default {
  name: 'SidebarItem',    //组件自调用，必须有name属性
  props: {                //接收父组件传参
    item: {
      type: Object,
      required: true
    },
    basePath: {     //从父组件一直拼接下来的基路径
      type: String,
      default: ''
    }
  },
  methods: {
    //拼接父子路径
    resolvePath(routePath) {
      return path.resolve(this.basePath,routePath)
    }
  }
}
</script>
```





### 项目展示

<div align=center><img src="https://raw.githubusercontent.com/Zimomo333/Vuex-Router-Webpack/master/picture/display.gif"></div>