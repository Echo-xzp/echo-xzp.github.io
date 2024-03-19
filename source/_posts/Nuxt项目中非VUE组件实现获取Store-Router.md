---
title: Nuxt项目中非VUE组件实现获取Store/Router
date: 2024-03-19 16:21:06
tags: [NuxtJS,nuxt,Vue,VueX,前端,JS,NodeJS,node]
categories: 前端
---

# 需求起源

楼主在做的一个`NuxtJS`项目中，发现前后端令牌`TOKEN`之间存在着过期时间不匹配的问题，也就是当我的后端`Redis`里面的令牌过期了之后，前端还存在着这个令牌，这个时候前端再去请求后端接口的时候就出错了，分析之后很快就找到了登录的时候设置了哪些缓存：

```javascript
  userLogin(this.loginObj).then(res => {
    this.subState = false
    this.$nuxt.$loading.finish()
     // 设置了TOKEN缓存 
    this.$store.commit('SET_TOKEN', res.token)
      
    this.$store.commit('GET_TEMPORARYURL')
      
    // 这里还设置了一些用户信息信息
    this.$store.dispatch('GET_USERINFO', store => {
      this.userInfo = this.$store.state.userInfo
      window.location.href = this.$store.state.temporaryUrl
    })
  }).catch((error) => {
    this.subState = false
    this.$nuxt.$loading.finish()
    this.$msgBox({
      content: error.msg,
      isShowCancelBtn: false
    })
  })
```

`GET_USERINFO`是`mutation.js`里面的方法，具体就是通过`TOKEN`又获取到了用户信息，然后存入`store`里面的`state`：

```javascript
  // 记录用户信息
  SET_USER: (state, data) => {
    state.tokenInfo = getStore('tokenInfo')
    state.userInfo = data
    setStore('OcUserInfo', data)
  }
```

具体代码写的比较散乱，就给不全了，大概就是这样就存在一个现象：**`TOKEN`已经过期了，但是用户信息还存在`store`里面**，于是前端认为用户还在登录状态，就会执行登录状态下的逻辑，但是从后端接口里面又获得不了什么数据，于是前端就有各种显示异常。

究其原因，解决这个问题其实很简单：

- 给`TOKEN`和`USER_INFO`*指定相同的过期时间*，要过期就都过期了，这种情况下按照原本前端的逻辑此时用户就处于未登录状态。
- 对`axios`返回结果进行处理，如果返回结果是用户令牌过期的错误提示，直接将用户缓存删除，也就是执行登出逻辑。

这里博主选择了第二种方法，这样处理感觉程序的健壮性更强，能够面对各种异常返回结果。于是博主就找封装和的`axios`方法，进行修改：

```javascript
import * as axios from 'axios'
import { Message } from 'element-ui'
import cookie from '../utils/cookies'
import config from '../config'

const createHttp = (token) => {
  const options = {}
  const head = {}
  
  //  获取store
  const store = this.$store
        
  if (process.server) {
    if (token) {
      head.token = token
    }
    options.baseURL = config.baseUrl
  } else {
    options.baseURL = '/gateway/'
  }
  if (process.client) {
    head.token = cookie.getInClient(config.tokenName)
  }
  options.headers = head
  const http = axios.create(options)
  http.interceptors.request.use(function(config) {
    return config
  }, function(error) {
    console.error(error)
    if (process.client) {
      return Promise.reject(error)
    }
  })

  http.interceptors.response.use(function(response) {
    const res = response.data
    if (res.code === 200) {
      return Promise.resolve(res.data)
    } else {
      if (process.client) {
        try {
          const d = JSON.parse(response.config.data || response.config.params || {})
          if (d.isShowErrTip !== false) {
              
             // 处理返回异常的结果
            if (res.code === 403 && res.msg === 'TOKEN_ERROR') {
               // 执行mutations的登出逻辑，其实就是把缓存都删除 
              store.commit('SIGN_OUT')
            }
            return Promise.reject(response.data)
          } else {
            return Promise.resolve(response.data)
          }
        } catch (error) {
          console.error(error)
          console.error(response.data)
          return Promise.resolve(response.data)
        }
      } else {
        return Promise.resolve(response.data)
      }
    }
  }, function(error) {
    if (process.client) {
      return Promise.reject(error)
    } else {
      console.error(JSON.stringify(error))
      return Promise.resolve(error.response.data)
    }
  })
  return http
}

export default createHttp
```

上面的代码其实就是加了那几个注释了的地方，但是运行起来直接就**报错**：`this.$store is undefined`

那就是拿不到的问题了，网上找了一下，才知道：

> **组件内未正确引用`Vuex`**：确保你正在尝试在已经安装了`Vuex`的组件中访问`this.$store`。如果在组件中使用`this.$store`仍然返回`undefined`，那么可能是因为该组件没有正确地引用`Vuex`。

也就是**必须在`VUE`的组件里面这样子调用才有用**，而我是在单独的一个`JS`文件里面，显然这样子就失效了。

# 需求分析

那么如何实现在非`VUE`组件实现获取`Store/Router`呢？博主前端水平有限，只能网上找找结果了，发现其实也是有很多人和我有一样的需求，找到一个好回答[Vue 在其他js文件中引入store的疑惑 ](https://segmentfault.com/q/1010000039132995)，大概意思就是通过`Vue`组件的方法拿不到，但是在`ES6`中已经模块化好了，那就直接把`store`导入不就行了？解决起来确实很简单啊，但是这种方法我一看我的项目似乎行不通啊：我的项目是`Nuxt`项目，`VUE`的装载都是他自行完成的，我的`store`并不是直接创立出来的，而是提供一个创立`store`的方法，交给`Nuxt`来创立，所以我就没法`import`进入我需要`store`的地方啊。

对比一下`store/index.js`两者的区别：

```javascript
// 纯VUE项目
import Vuex from 'vuex'
import actions from './actions'
import mutations from './mutations'
import config from '../config'

const store = 
   new Vuex.Store({
    state: () => ({
      tokenName: config.tokenName,
      tokenInfo: '', // token信息
      temporaryUrl: '', // 临时url
      websiteInfo: null, // 站点信息
      videoUrl: '' // 视频url
    }),
    mutations: mutations,
    actions: actions
  })

export default store
```

```javascript
// Nuxt项目里的
import Vuex from 'vuex'
import actions from './actions'
import mutations from './mutations'
import config from '../config'

const createStore = () => {
  new Vuex.Store({
    state: () => ({
      tokenName: config.tokenName,
      tokenInfo: '', // token信息
      temporaryUrl: '', // 临时url
      websiteInfo: null, // 站点信息
      videoUrl: '' // 视频url
    }),
    mutations: mutations,
    actions: actions
  })
}

export default createStore
```

我要是导入了只能获得`createStore`这个方法啊，再次调用只会产生新的`store`，而不是获取原来的`store`吧，显然达不到我的需求。

那只能继续找找其他方法，于是又找到一篇博客：[nuxtjs如何在单独的js文件中引入store和router](https://www.cnblogs.com/goloving/p/11440880.html)

但是他说的方法我的失败了：

- 用插件的方法对我这种二把刀来讲貌似有点复杂，就没有尝试。
- 导入`store`这种方法就是上面分析的那样，导入试了也没有啥用，还是`undefined`
- 通过`windows`获取`$nuxt`进而获取`store`的方法运行起来直接报错，网上找了一下原因应该是`nuxt`只能初始化完成时才能获取。

那么似乎一下就进入死胡同里面了。

# 需求解决

但是回到最初的那种导入的方法，为啥不能用？就是上面说的现在只有一个`createStore`方法暴露给我用，用了也只会产生新的`store`，而不是原来那个`store`。这个时候忽然有点熟悉的感觉，这不就是`Java`里面的单例获取一样吗？我通过`getIntance`方法获取实例，每次调用只会获取原来已经创立的那个实例，反过来说，我把`createStore`转换成单例模式不就好了？这样我导入这个方法再调用，不就获得了`store`了吗？似乎一下就开阔了。写个闭包直接实现：

```javascript
import Vuex from 'vuex'
import actions from './actions'
import mutations from './mutations'
import config from '../config'

let store

const createStore = () => {
  return store || (store = new Vuex.Store({
    state: () => ({
      tokenName: config.tokenName,
      tokenInfo: '', // token信息
      temporaryUrl: '', // 临时url
      websiteInfo: null, // 站点信息
      videoUrl: '' // 视频url
    }),
    mutations: mutations,
    actions: actions
  }))
}

export default createStore
```

再修改我需要`store`的地方：

```javascript
import * as axios from 'axios'
import { Message } from 'element-ui'
import cookie from '../utils/cookies'
import config from '../config'
// 直接导入
import createStore from '@/store'

const createHttp = (token) => {
  const options = {}
  const head = {}
  
  // 获取 store
  const store = createStore()
  
  if (process.server) {
    if (token) {
      head.token = token
    }
    options.baseURL = config.baseUrl
  } else {
    options.baseURL = '/gateway/'
  }
  if (process.client) {
    head.token = cookie.getInClient(config.tokenName)
  }
  options.headers = head
  const http = axios.create(options)
  http.interceptors.request.use(function(config) {
    return config
  }, function(error) {
    console.error(error)
    if (process.client) {
      return Promise.reject(error)
    }
  })

  http.interceptors.response.use(function(response) {
    const res = response.data
    if (res.code === 200) {
      return Promise.resolve(res.data)
    } else {
      if (process.client) {
        try {
          const d = JSON.parse(response.config.data || response.config.params || {})
          if (d.isShowErrTip !== false) {
              
             // 处理返回异常的结果
            if (res.code === 403 && res.msg === 'TOKEN_ERROR') {
               // 直接使用 store
              store.commit('SIGN_OUT')
            }
            return Promise.reject(response.data)
          } else {
            return Promise.resolve(response.data)
          }
        } catch (error) {
          console.error(error)
          console.error(response.data)
          return Promise.resolve(response.data)
        }
      } else {
        return Promise.resolve(response.data)
      }
    }
  }, function(error) {
    if (process.client) {
      return Promise.reject(error)
    } else {
      console.error(JSON.stringify(error))
      return Promise.resolve(error.response.data)
    }
  })
  return http
}

export default createHttp
```

直接测试，`Redis`删除令牌，发现只要给后端发请求错误一次，直接就将用户登出了，完美解决。但是实际上这种方法可能还存在隐患，会不会和`Java`的单例那样存在线程同步的问题，进而要加锁解决，具体应该再测试一下并完善一下。

同理参考这种方法，应该也很简单能实现获取`Router`的方法。
