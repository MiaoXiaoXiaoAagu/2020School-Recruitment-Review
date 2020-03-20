## 主要原理：

### 1.为什么用hash、history

[vue-router的两种模式的区别](https://juejin.im/post/5a61908c6fb9a01c9064f20a)

前端路由的原理就是不向后端发送请求，根据`url`来匹配不同的组件

`hash`不会发送`http`请求，`history`操作浏览器历史栈，也不会。

### 2.hash、history的使用

`hash`的改变对应的是`hashchange`的事件

```js
 window.addEventListener('hashchange', () => {})
```

`history`的改变对应的是`popstate`

```js
 window.addEventListener('popstate', () => {})
```

### 3.Router原理

监听`url`的改变触发`current`改变——>`current`在vue中是响应式的，会触发视图的重新渲染——>根据`current`和`router`的传入参数匹配组件，`render`渲染组件。

## 代码结构分析

### 1.插件必备的install方法中需要作什么

`VueRouter`是`Vue`的插件，插件需要有`install`方法。`Vue.use`会调用插件的`install`方法，并且传入Vue。暴露`Vue`给插件，并且`install`插件实现插件的安装。

`install`通过`mixin()`混入到`beforeCreate`中

​		1.使Vue获得`$router`、`$route`

​		2.使`current`在Vue中变为响应式，以便`current`的改变触发路由的重新渲染。

### 2.注册router-view

`<router-view></router-view>`是路由相关组件的占位符，`<chid-comp></chid-comp>`只是`ChildComp`组件的占位符

router-view通过`render`方法根据url选择不同的组件渲染

以下为render的使用方法：[渲染函数](https://cn.vuejs.org/v2/guide/render-function.html)

```js
 Vue.component('router-view', {
    render (h) {
      return h(组件或者)
    }
  })
```

### 3.监听hash、history

监听`hash`、`history`——>修改`current`,`current`在`vue`中已经注册`defineReactive`注册为响应式。

**hash对应的地址：**`location.hash`

​					**监听：**`hashchange`

**history对应的地址：**`loaction.pathname`

​						**监听：**`popstate`



## 源码：

[珠峰架构:手写vue-router源码]([https://libin1991.github.io/2019/11/01/%E7%8F%A0%E5%B3%B0%E6%9E%B6%E6%9E%84-%E6%89%8B%E5%86%99vue-router%E6%BA%90%E7%A0%81/](https://libin1991.github.io/2019/11/01/珠峰架构-手写vue-router源码/))

我手写的源码：（不包含router-link）

```js
/* eslint-disable */
class HistoryRoute {
  constructor () {
    this.current = null
  }
}
class vueRouter {
  constructor (options) {
    this.mode = options.mode || 'hash'
    this.history = new HistoryRoute()
    this.routes = options.routes || []
    this.routesMap = this.createRouteMap()
    this.init()
  }

  init () {
    if (this.mode === 'hash') {
      location.hash ? '' : location.hash = '/'
      window.addEventListener('load', () => {
        this.history.current = location.hash.slice(1)
      })
      window.addEventListener('hashchange', () => {
        this.history.current = location.hash.slice(1)
      })
    } else {
      location.pathname ? '' : location.pathname = '/'
      window.addEventListener('load', () => {
        this.history.current = location.pathname
      })
      window.addEventListener('popstate', () => {
        this.history.current = location.pathname
      })
    }
  }
  createRouteMap () {
    return this.routes.reduce((map, current) => {
       map[current.path] = current.component
      return map
    }, {})
  }
}
vueRouter.install = function (Vue) {
  Vue.mixin({
    beforeCreate () {
      if (this.$options && this.$options.router) {
        this._root = this
        this._router = this.$options.router
        this._route = this._router.history
        Vue.util.defineReactive(this, 'xxx', this._router.history)
      } else {
        this._root = this.$parent._root
      }
      Object.defineProperty(this,'$router',{
          get () {
            return this._root._router
        }
      })
      Object.defineProperty(this,"$route",{
        get () {
          return this._root._router.history
        }
      })
    }
  })

  Vue.component('router-view', {
    render (creatElement, hack) {
      return creatElement(this.$router.routesMap[this.$route.current])
    }
  })
}
export default vueRouter

```

**注意点：**

**1._root的作用**

`new vueRouter`的实例是从根节点获取的，`Vue.mixin`是对所有节点的操作，要让所有节点都能获取到`$router`、`$route`就要从根节点取到`new vueRouter`，所以有必要让每一个节点都取到根节点`._root`

```js
Vue.mixin ({
    beforeCreate () {
        if (this.$options && this.$options.router){
            this._root=this
		} else {
          this._root=this.$parent._root  
        }
    }
})
```



**2.`Object.defineProperty`定义`$router`、`$route`**

已经可以通过`_root`获取到`"$router"`与`"$route"`,为什么还要定义出来

`Object.defineProperty`通过属性描述符可以控制`$router`与`$route`的只读



**3.return map[key] = obj**

此时return的返回值是obj而不是map