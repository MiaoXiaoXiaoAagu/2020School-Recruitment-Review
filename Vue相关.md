# 刷题：

[30 道 Vue 面试题，内含详细讲解](https://juejin.im/post/5d59f2a451882549be53b170)

[2020年大厂面试指南 - Vue篇](https://juejin.im/post/5e4d24cce51d4526f76eb2ba#comment)

[Vuex面试题汇总](https://juejin.im/post/5dba91e4518825647e4ef18b)

# 基础

[2020年大厂面试指南 - Vue篇](https://juejin.im/post/5e4d24cce51d4526f76eb2ba#comment)

### 组件通信方式有哪些？

​	[vue 组件通信看这篇就够了(12种通信方式)](https://mp.weixin.qq.com/s?__biz=Mzg2NTA4NTIwNA==&mid=2247484468&idx=1&sn=3c309945992fe4f0c6276d91d7cb67b8&chksm=ce5e364ff929bf59ae686e3fa2412632348d6e190054e0b83bed11b5fd851ebbe8d326b5caf0&token=1424393752&lang=zh_CN#rd)

### 父子组件之间的通信：

	#### 		1.prop 和 events:

​			[Vue组件基础]([https://cn.vuejs.org/v2/guide/components.html#%E7%9B%91%E5%90%AC%E5%AD%90%E7%BB%84%E4%BB%B6%E4%BA%8B%E4%BB%B6](https://cn.vuejs.org/v2/guide/components.html#监听子组件事件))

​			props:[variance]子组件接收数据,父组件v-bind:variance传参。

​			子组件.$emit()触发事件并传递参数，父组件监听

​			**注意：**Vue中的规范是使用单向数据流，不要在子组件内部改变prop，要对prop有所改变再使用，用computed。prop传递基本类型，子组件改变不会对父组件影响，但是如果是引用类型就会产生影响，违反单向数据流原则。





	#### 		2.v-model:

​			 [v-model的基础用法](https://cn.vuejs.org/v2/guide/forms.html#基础用法)

​			**基础用法：**`v-model`是`v-bind`与`v-on`的语法糖，`v-model='variance'`,相当于`v-bind:value=variance`,v-`on:input=updateVriance`,updateVriance()负责更新`variacne`为最新输入框中值。

​			**v-model在组件中的使用：**相当于将子组件看作是一个Input输入框，不过子组件要自己`$emit('input',data)`，并且要`props:['value']`。如对Input框使用v-model一样，父组件自动添加对`input`事件的监听，处理函数将v-model值改为函数传递值。



#### 		3.sync 修饰符：

​			[`.sync` 修饰符](https://cn.vuejs.org/v2/guide/components-custom-events.html#sync-修饰符)

​			**update:myPropName模式：**子组件进行`update:myPropName`的形式进行触发函数例如：`this.$emit('update:title', newTitle)`

父组件:

```html
<text-document
  v-bind:title="doc.title"
  v-on:update:title="doc.title = $event"
></text-document>
```

以上可以用.sync修饰符缩写为`v-bind:title.sync="doc.title"`

​		`	.sync`修饰符的作用就是为了实现如`v-model`一样的子组件改变父组件的值，所以整个流程需要的就是1.传递值  2.改变值    `update:prototypeName`模式,值属性名为title,那么emit事件就为`update:title`。并且由于update:title的处理函数就只是title=newTitle,也没有用户手写的必要。

​		所以`.sync`修饰符对于父组件而言像是普通的`v-bind:title.sync`,对于子组件`$emit('update:title',newTtile)`





	#### 		4.ref:

​			ref如同组件的一个`ID`  父组件使用子组件时标注`ref=‘refID’`，在父组件中`$this.refs.refID`就可以获取这个子组件上的数据、方法

​			**注意：**1.`$refs` 是作为渲染结果被创建的，所以在初始渲染的时候它还不存在，此时无法无法访问。

​						2.`$refs` 不是响应式的，只能拿到获取它的那一刻子组件实例的状态，所以要避免在模板和计算属性中使用它。





	#### 		5.$parent、$children

​				**$parent:**子组件的直接父组件

​				**$children:**父组件的直接子组件**们**，且不保证顺序。

------



### 非父子组件通信

#### 		1.$attrs 和 $listeners

​			父、孙组件之间传递

​			[Vue 父子组件数据传递( inheritAttrs + $attrs + $listeners)](https://juejin.im/post/5ae4288a5188256712784787)

​			**$attrs：**把父组件的数据传递给孙组件。父组件绑定数据data1,data2,data3到子组件中，子组件的`props`只接收了`['data1']`,那么data2、data3就可以传递到孙组件中`$attrs={data2,data3}`

​			父—>子:父组件使用childComp子组件，传递data1、data2、data3到子组件中

```html
<childComp :data1='D1',:data2='D2',:data3='D3'></childComp>
```

​			子—>孙:子组件将自己收到的，除了在props中使用的属性都传给孙

```html
<grandComp v-bind="$attrs"></grandComp>
```

```js
props:['data1']
```

​			孙：使用子组件作为中间传递者过滤过来的数据

```html
<p>{{$attrs}}</p>//{data2:D2,data3:D3}
```



​			**$listeners:**子组件通过`v-on=‘$listeners’`让孙组件的`$emit(`grandEvent`)`可以被父组件的`v-on:'grandEvent'='grandHandler'`监听到



#### 2.provide 和 inject

祖先组件与子孙组件之间的通讯方式,可以传递数据、方法

**注意：**一般不用在业务中，用于组件库与高级组件

```js
<!--祖先组件-->
<script>
export default {
    provide: {
        author: 'yushihu',
    },
    data() {},
}
</script>
```

```js

<!--祖先组件-->
<script>
export default {
    provide: {
        author: 'yushihu',
    },
    data() {},
}
</script>
```



#### 3.event bus

任意两个组件之间的通讯，**集中式的事件中间件**

[Vue 组件通信之 Bus](https://juejin.im/post/5a4353766fb9a044fb080927)

Vue中注入`$bus`

```js
var eventBus = {
    install(Vue,options) {
        Vue.prototype.$bus = vue
    }
};
Vue.use(eventBus);
```

触发事件：`this.$bus.$on('eventNanme',handler)` 

监听:`this.$bus.$emit('eventName',value)`  

清除事件监听：`this.$bus.$off('eventName')`



#### 4.通过 $root 访问根实例

所有组件都可以通过$root访问组件的根实例，维护根实例的data,可以实现任意组件间通信



#### 5.简单的store模式

[简单状态管理起步使用](https://cn.vuejs.org/v2/guide/state-management.html#简单状态管理起步使用)

需要数据的共享的组件引入同一个js对象`store`作为数据源。为了如同VUEX一样数据状态的改变便于管理，所有的数据变化都通过`store`中的函数，并且`set`改变数值与`clear`清空值得方法调用都要打印日志。

```js
var store = {
  debug: true,
  state: {
    message: 'Hello!'
  },
  setMessageAction (newValue) {
    if (this.debug) console.log('setMessageAction triggered with', newValue)
    this.state.message = newValue
  },
  clearMessageAction () {
    if (this.debug) console.log('clearMessageAction triggered')
    this.state.message = ''
  }
}
```

```js
var vmA = new Vue({
  data: {
    privateState: {},
    sharedState: store.state
  }
})

var vmB = new Vue({
  data: {
    privateState: {},
    sharedState: store.state
  }
})
```



#### 生命周期：

[vue生命周期钩子函数的正确使用方式](https://www.jianshu.com/p/a20f2023c78a)

[Vue.js 技术揭秘|生命周期](https://ustbhuangyi.github.io/vue-analysis/)

![img](.\img\10)

**beforeCreate():**在Vue进行一些初始化之后，与method、data、prop等数据无关。Vue实例基本是空的，但是`$router`有了，可以进行重定向。

**created():**在initState()之后,Vue上的method、data、prop之类的初始化好了。但是`$el`还没有

**beforeMount()**:render()好了，但是Vnode还没有

**mounted()**:渲染好了

#### Vue 的父组件和子组件生命周期钩子执行顺序

​	**渲染：**父组件在挂载的时候才会把子组件的占位符换为component并且渲染子组件

​	父beforeCreate -> 父created -> 父beforeMount -> 子beforeCreate -> 子created -> 子beforeMount -> 子mounted -> 父mounted

​	**子组件更新过程：**

影响到父组件： 父beforeUpdate -> 子beforeUpdate->子updated -> 父updted

不影响父组件： 子beforeUpdate -> 子updated



**父组件更新过程：**

影响到子组件： 父beforeUpdate -> 子beforeUpdate->子updated -> 父updted

不影响子组件： 父beforeUpdate -> 父updated





#### v-show 和 v-if 有哪些区别？

v-if=false初始化不会渲染也不会订阅数据，v-show会初始化渲染。

v-if销毁节点、新增节点。用于切换的开销大

v-show只是进行css的变化。初始化开销大



#### keep-alive的作用

缓存其包裹的组件而不是销毁。

组件被缓存时执行deactived,命中缓存时执行actived

#### 为什么Vue3.0不再使用defineProperty实现数据监听？

[Proxy](https://es6.ruanyifeng.com/#docs/proxy)

[【深入vue】为什么Vue3.0不再使用defineProperty实现数据监听？](https://mp.weixin.qq.com/s?__biz=Mzg2NTA4NTIwNA==&mid=2247485104&idx=1&sn=648f1850fb59f04c6ca52cdd794013b5&chksm=ce5e34cbf929bddd4325de7f78dc4ad5822cddff69c999d1225f860e917dfd3d6eea43527c56&token=1424393752&lang=zh_CN#rd)

因为Proxy是对`obj.`点操作运算符、`obj['key']`，`for... in`等操作运算进行拦截，而不是像`Object.defineProperty`对`getter、setter`

就不存在新增加的值没有`Observe`的情况



####  组件的 data 为什么要写成函数形式

为了实现组件的复用，一个组件在多处使用维护的是不同的数据。

`event bus`这样使用同一份数据的则不需要返回一个函数

#### 

#### Vue 的模板编译原理

[Vue 模板编译原理](https://github.com/berwin/Blog/issues/18#)

解析模板字符串，生成AST树

`Stack`栈记录`dom`层级关系

模板字符串在while循环中一段一段的截取，`<div>`、`dsadsdfsd`、`</div>`  标签头部、字符串、标签尾部各成一段。

```html
<div>
  <p>{{name}}</p>
</div>
```

```html
  <p>{{name}}</p>
</div>
```

```html
<p>{{name}}</p>
</div>
```

```html
{{name}}</p>
</div>
```

```html
</p>
</div>
```

```html
</div>
```

​	`stack`栈顶永远都是正在解析的节点的`parentNode`父节点，将正在解析的节点`push`到栈顶的`parentNode.children`中——>构建出dom树的层级关系。如果是元素节点，再`push`到栈顶。正在解析的节点可能是文本节点。

​	如果一个节点不是文本节点也可能不`push`到`stack`中。例如`input`是自闭和节点，自闭和节点是没有子节点的，就没有必要`push`到`stack`中。

​	**注意：**文本节点中可能带有`<`会被误判成开始标签

​				**解决方式：**见[Vue 模板编译原理](https://github.com/berwin/Blog/issues/18#)。

**{{模板字符串}}：**解析到的文本节点如果是纯文本节点就直接`push`到`parentNode.children.text`中。如果包含模板字符串则解析出模板字符串放到`parentNode.children.express`中

**结束标签：**截取到结束标签，在stack中从后往前匹配，匹配到之后把其后面的（包括自身）所有标签删除。因为在stcak中，闭合的标签之后的标签肯定是自身的子、孙级关系的标签，自己都闭合了，那么子标签也是闭合了的（或者就是写错了，没有闭合的标签）。

#### v-for中key的作用是什么

 为了在diff算法中迅速判断两个相似的节点是不同节点



#### v-for为什么不与if同时使用

因为`if=false`对标签、组件进行不渲染，这样的话不如对`v-for`数据在`computed`中进行过滤，一开始就不要出现在`v-for`数据中



#### v-router导航守卫

[vue-router导航守卫，不懂的来](https://zhuanlan.zhihu.com/p/54112006)

**三类：**全局守卫、单个路由独享守卫、组件守卫

**全局守卫：**beforeEach、beforeResolve、afterEach

**单个路由独享守卫：**beforeEnter

**组件守卫：**beforeRouteEnter、beforeRouteUpdate、beforeRouteLeave

​		**规律总结：**全局守卫带each、路由守卫带enter、组件守卫带route。beforeResolve是针对的异步组件



**顺序：**beforeEach、beforeEnter、beforeRouteEnter、beforeResolve、afterEach

![preview](.\img\v2-c3a67a5eb0b8da4936a6b57ef8c48783_r.jpg)

​			**规律总结：**基本顺序为：全局守卫、路由守卫、组件守卫

`beforeResolve`针对异步组件，所以在before中排最后。`afterEach`全局后置守卫，所以在`beforeResolve`之后。

​	

------

#### computed与watch的区别

#### computed与watch的实现

#### nextTick是做什么的，原理是什么

## vuex相关

[Vuex 通俗版教程](https://www.jianshu.com/p/caff7b8ab2cf)

[Vuex面试题汇总](https://juejin.im/post/5dba91e4518825647e4ef18b#heading-31)

![vuex](.\img\vuex.png)

#### action与mutation的区别：

​		1.action提交的是mutation而不是直接改变状态

​		2.action是异步操作、mutation是同步操作。mutation每次改变state都会有一个快照,如果mutation是异步，devtools调试工具就摸不着头脑了。

​		3.mutation的第一个参数是state,action是context,context对象内容如下

```js
{
    state,      // 等同于 `store.state`，若在模块中则为局部状态
    rootState,  // 等同于 `store.state`，只存在于模块中
    commit,     // 等同于 `store.commit`
    dispatch,   // 等同于 `store.dispatch`
    getters,    // 等同于 `store.getters`
    rootGetters // 等同于 `store.getters`，只存在于模块中
}

```

​	

#### action通常都是异步的，怎么知道action什么时候完成

action返回promise,`this.$store.dispatch`调用action之后`.then()`

```js
actions:{
    SET_NUMBER_A({commit},data){
        return new Promise((resolve,reject) =>{
            setTimeout(() =>{
                commit('SET_NUMBER',10);
                resolve();
            },2000)
        })
    }
}
this.$store.dispatch('SET_NUMBER_A').then(() => {
  // ...
})
```



#### 模块总结：

​	在未设置`namespace=true`时，除了state是局部模块独有的，action、mutation、**getter**都是注册在**全局命名空间的**

**命名空间`namespace=true`时：**

​	模块中`mutation`的参数`state,rootState`

​				`getter`的参数`state、getter、rootState、rootGetter`

​	**`mutation`只能对本地`state`修改**

​	action中context：

```js
{
    state,      // 等同于 `store.state`，若在模块中则为局部状态
    rootState,  // 等同于 `store.state`，只存在于模块中
    commit,     // 等同于 `store.commit`
    dispatch,   // 等同于 `store.dispatch`
    getters,    // 等同于 `store.getters`
    rootGetters // 等同于 `store.getters`，只存在于模块中
}
```



**使用局部模块中的getter、mutation、action**

gettter:`getters['account/isAdmin']`

action:`dispatch('account/login')`

mutation:`commit('account/login')`

```js
const store = new Vuex.Store({
  modules: {
    account: {
      namespaced: true,

      // 模块内容（module assets）
      state: { ... }, // 模块内的状态已经是嵌套的了，使用 `namespaced` 属性不会对其产生影响
      getters: {
        isAdmin () { ... } // -> getters['account/isAdmin']
      },
      actions: {
        login () { ... } // -> dispatch('account/login')
      },
      mutations: {
        login () { ... } // -> commit('account/login')
      }
   }
}          
```



#### v-model怎么使用Vuex中的state

**重点：**避免`v-model`自带的函数修改Vuex中的state

**方法：**使用`computed`中的`setter`,当computed被修改时会触发相应`setter`,在`setter`中进行`.$store.commit()`



#### Vuex的严格模式：

只要`state`不由`mutation`修改就会报错，这样确保了`state`的所有变化都由`mutation`记录了`log`日志，便于调试



#### Vuex的生存时间？？

Vuex只要一刷新就失效，因为浏览器的机制，js的所有数据都是放在堆栈中的，每当浏览器刷新则清除这些数据。

Vuex 失效解决方法：插件、localstorage  [解决Vuex刷新页面数据丢失问题 ---- vuex-persistedstate持久化数据](https://www.cnblogs.com/ljx20180807/p/10827250.html);[vuex页面刷新数据丢失的解决办法](https://juejin.im/post/5c809599f265da2dbe030ec6)