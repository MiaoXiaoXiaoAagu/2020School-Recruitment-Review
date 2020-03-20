# 手写Promise

参考：[Promise实现原理（附源码）](https://juejin.im/post/5b83cb5ae51d4538cc3ec354)

## promise的注意点：

1.状态不可变

2.值穿透，`then`的回调`resolve`要是不是一个`function`而是基本类型，那么这个基本类型就会传递给下一个`then`	[关于promise的一道面试题](https://segmentfault.com/q/1010000012002040)

3.`then`的回调`resolve `中`throw erro`会跳到下一个`then`回调的`reject`中



## 高阶函数：接收函数作为参数的函数

[高阶函数——廖雪峰](https://www.liaoxuefeng.com/wiki/1022910821149312/1023021271742944)

**1.一个最简单的高阶函数：**

```js
function add(x, y, f) {//add为高阶函数
    return f(x) + f(y);
}
```



**2.高阶函数决定了回调的调用时间以及接收的参数**

例如：`高阶函数fn`接收函数callBack作为参数,就决定了callBack的调用时间以及callBack的参数

调用fn时定义callBack

```js
function fn(callBack) {//定义一个有回调的function——》fn
    //fn决定何时调用这个回调，以及对回调的传参
    callBack(1,2)
}

fn(function (fir,sec) {//调用fn传入回调，决定回调拿到传参的具体执行
    return fir+sec
})
```



**3.高阶函数的回调接收的参数仍然是函数**

例如：

定义高阶函数fn，并且给fn的回调传入函数

调用fn时决定回调的具体内容，以及何时使用fn给的`函数参数resolve、reject`

```js
function fn(callBack) {//定义一个有回调的function——》fn
    //fn决定何时调用这个回调，以及对回调的传参
    let resolve=function(){
        console.log('resolve')
    }
    let reject=function(){
        console.log('reject')
    }
    
    callBack(resolve,reject)//对回调callBack的传参为函数
    
}

fn(function (resolve,reject) {//调用fn传入回调，决定回调拿到传参的具体执行
    return resolve()//回调的定义中决定什么时候使用收到的传参
})

```



## Promise架构：

### **1.new Promise().then()**

​		**高阶函数的使用:**

`new MyPromise(handle)`的回调——>handles收到的参数为函数

`new MyPromise`会执行`calss MyPromise`的构造函数,所以class MyPromise是在定义高阶函数，并且回调接收到的参数也就是函数（`resolve、reject`）。

```js
class MyPromise{//定义
    constructor(handle){
        handle(this.res,this.rej)
    }
    res(){}

    rej(){}
}
new MyPromise(function (resolve,reject) {//使用
    resolve(1)
}).then()
```

`new MyPromise().then()`表示执行`MyPromise`的构造函数之后立马执行`then()`		

**then根据MyPromise实例使用的是resolve()还是reject()来选择回调**

所以`resolve(),reject`改变`MyPromise`中的`this.state`，then再根据`this.state`选择执行`resolve`还是`reject`



此步骤完成代码如下：

```js
const PENDING = 'PENDING'//进行中
const FULFILLED = 'FULFILLED'//已成功
const REJECTED = 'REJECTED' //已失败

class MyPromise{
    constructor(handle){
        this._value = null
        this._status = PENDING
        handle(this._resolve.bind(this),this._reject.bind(this))
    }
    _resolve(val){
        if (this._status !== PENDING) return//状态不可逆，只能从PENDING——》FULFILLED，不能从别的状态到FULFILLED
        this._value=val
        this._status=FULFILLED
    }

    _reject(val){
        if (this._status !== PENDING) return//状态不可逆，只能从PENDING——》REJECTED，不能从别的状态到REJECTED
        this._value= val
        this._status=REJECTED
    }
}

MyPromise.prototype.then=function (onFulfilled,onRejected) {
    switch (this._status) {
        case PENDING:break;
        case FULFILLED:onFulfilled(this._value);break;
        case REJECTED:onRejected(this._value);break;
    }

}

new MyPromise(function (resolve,reject) {
    resolve('call resolve')
}).then(function (val) {
    console.log(val)
})//call resolve

new MyPromise(function (resolve,reject) {
    reject('call reject')
}).then(function () {},function (val) {
    console.log(val)
})//call reject

```

​				**注意：1. 高阶函数对this的处理：**`handle(this._resolve.bind(this),this._reject.bind(this))` 传递`_resolve`与`_reject`给`handle`调用，那么它们的`this`会随着`hangle`走，所以此处应该要`bind`

​							**2.Promise的状态不可变:**由于只有`_resolve`与`_reject`会修改状态，所以只要保证`_resolve`

、`_reject`修改状态前状态都为`	PENDING`就可以做到状态不可变

​																

### **2.异步操作**

​		`new MyPromise(handle1).then(handle1)`是同步的，`handle1`执行之后会立刻执行`then()`,异步操作到任务队列中等待执行。但此时`handle1`的异步操作还没有执行，没有进行`resolve`、`reject`所以`then`中的回调也无法执行。

​		此时引入`Pending（进行中）、Fulfilled（已成功）、Rejected（已失败）`状态，`resolve()`可以将`Pending`转变为`Fulfilled`,同理`reject()`将`Pending`转变为`Rejected`状态

​		`new MyPromise(handle1).then(handle1)`执行到`then()`时如果有异步操作，那么状态仍为`Peding`，此时将`then`的回调都储存起来，等待`resolve()、reject()`执行时再执行。



**此步骤完成代码如下：**

```js
const PENDING = 'PENDING'//进行中
const FULFILLED = 'FULFILLED'//已成功
const REJECTED = 'REJECTED' //已失败

class MyPromise{
    constructor(handle){
        this._value = null
        this._status = PENDING
        // 添加成功回调函数队列
        this._fulfilledQueues = []
        // 添加失败回调函数队列
        this._rejectedQueues = []
        handle(this._resolve.bind(this),this._reject.bind(this))
    }
    _resolve(val){
        if (this._status !== PENDING) return//状态不可逆，只能从PENDING——》FULFILLED，不能从别的状态到FULFILLED
        this._value=val
        this._status=FULFILLED
        let cb
        while (cb=this._fulfilledQueues.shift()){//依次执行成功队列中的函数，并清空队列
            cb(val)
        }
    }

    _reject(val){
        if (this._status !== PENDING) return//状态不可逆，只能从PENDING——》REJECTED，不能从别的状态到REJECTED
        this._value= val
        this._status=REJECTED
        let cb
        while (cb=this._rejectedQueues.shift()){//依次执行失败队列中的函数，并清空队列
            cb(val)
        }
    }
}

//使用例子
MyPromise.prototype.then=function (onFulfilled,onRejected) {
    switch (this._status) {
        case PENDING: this._fulfilledQueues.push(onFulfilled)//待选择，待执行的回头添加到队列
                      this._rejectedQueues.push(onRejected)
                      break
        case FULFILLED: onFulfilled(this._value);break;
        case REJECTED: onRejected(this._value);break;
    }

}

new MyPromise(function (resolve,reject) {
    setTimeout(function () {
        resolve('call resolve')
    },1000)

}).then(function (val) {
    console.log(val)
})//1秒之后输出 call resolve

new MyPromise(function (resolve,reject) {
    setTimeout(function () {
        reject('call reject')
    },2000)

}).then(function () {},function (val) {
    console.log(val)
})//2秒之后输出 call reject

```



### **3.链式调用**

#### 3.1链式

​		定义`then`方法时:

​									让`then`返回一个`MyPromise`对象可实现链式调用。

`onFullfilled(val)`或`onRejectedNext(val)`的调用决定了`new MyPromise().then().then(res,rej)`中的第二个then的回调什么时候调用，收到的传参是多少。

```json
MyPromise.prototype.then(function(onFullfilled,onRejected){
    return new MyPromise(function(onFullfilledNext,onRejectedNext){
        onFullfilledNext(val)//onRejectedNext(val)
	})
})
```

#### 3.2链式与异步

常见情况下，`onFullfilledNext`的调用时间取决于`onFullfilled`的调用时间，`onFullfilledNext(val)`传递的参数`val`是`onFullfilled`的返回值

但是在异步情况下，`onFullfilled`是传入`._fulfilledQueues`队列中等待执行的,所以将`onFullfilled`打包在`fulfilled`中延迟调用，将`fulfilled`代替`onFullfilled`放入队列。`fulfilled`中`onFullfilledNext`根据`onFullfilled`的返回值传参。

**此步骤代码：**

```js
function isFunction(fn) {
    return typeof fn === 'function'
}
const PENDING = 'PENDING'//进行中
const FULFILLED = 'FULFILLED'//已成功
const REJECTED = 'REJECTED' //已失败

class MyPromise{
    constructor(handle){
        this._value = null
        this._status = PENDING
        // 添加成功回调函数队列
        this._fulfilledQueues = []
        // 添加失败回调函数队列
        this._rejectedQueues = []
        try{
            handle(this._resolve.bind(this),this._reject.bind(this))
        }catch (err) {
            this._reject(err)
        }

    }
    _resolve(val){

        const run=() => {
            if (this._status !== PENDING) return//状态不可逆，只能从PENDING——》FULFILLED，不能从别的状态到FULFILLED
            // 依次执行成功队列中的函数，并清空队列
            const runFulfilled = (value) => {
                let cb;
                while (cb = this._fulfilledQueues.shift()) {
                    cb(value)
                }
            }

            // 依次执行失败队列中的函数，并清空队列
            const runRejected = (error) => {
                let cb;
                while (cb = this._rejectedQueues.shift()) {
                    cb(error)
                }
            }

            if(val instanceof MyPromise){
                val.then((value)=>{
                    this._value=value
                    this._status=FULFILLED
                    runFulfilled(value)
                },(err)=>{
                    this._value=err
                    this._status=REJECTED
                    runRejected(err)
                })
            }else {
                this._value=val
                this._status=FULFILLED
                runFulfilled(val)
            }
        }
        setTimeout(run,0)

    }

    _reject(err){
        if (this._status !== PENDING) return//状态不可逆，只能从PENDING——》REJECTED，不能从别的状态到REJECTED

        this._value= err
        this._status=REJECTED
        let cb
        while (cb=this._rejectedQueues.shift()){//依次执行失败队列中的函数，并清空队列
            cb(err)
        }
    }
}

MyPromise.prototype.then=function (onFulfilled,onRejected) {
    return new MyPromise( (onFulfilledNext,onRejectedNext) => {
        let fulfilled=(value) => {
            try {
                let result = onFulfilled(value)
                    onFulfilledNext(result)
            }catch (err) {
                onRejectedNext(err)
            }
        }

        let rejected=(value) => {
            try{
                let result =onRejected(value)
                    onRejectedNext(result)
            }catch (err) {
                onRejectedNext(err)
            }
        }

        switch(this._status){
            case PENDING : this._fulfilledQueues.push(fulfilled)
                this._rejectedQueues.push(rejected)
                break
            case FULFILLED :fulfilled(this._value)
                break
            case REJECTED : rejected(this._value)
                break
        }
    })

}

//使用例子
new MyPromise(function (resolve,reject) {
    setTimeout(function () {
        resolve(new MyPromise(function (resolve) {
            resolve('promise in resolve')
        }))
    },1000)

}).then(function (val) {
    console.log('first then,get message :'+val)//1秒之后输出 first then,get message :promise in resolve
    return 'val form 1st then to 2nd then'
}).then(function (val) {
    console.log('second then,get message:'+val)//1秒之后输出 second then,get message:val form 1st then to 2nd then
})

```

#### 3.3值穿透与`then`的回调返回`MyPromise`对象

**1.onFullfilled——》对应调用onFullfilledNext**

`onFullfilled`也就是第一个`then`的`res`回调

​				`onFullfilled`不为function时：值穿透，直接`onFullfilledNext(onFullFiled)`

​				`onFullfilled`为function时：1.如果返回值不为Promise，`onFullfilledNext(返回值)`

​																	2.如果返回值为Promise时，`返回值.then(onFullfilled)`。将`onFullfilledNext`传给then当回调，由`Promise`决定何时将执行权传递给`then`

​		

**2.onRejected——》对应调用onRejectedNext**

`onRejected`也就是then的`rej`回调

​				`onRejected`不为`function`时：值穿透，直接调用`onRejectedNext(onRejected)`

​				`onRejected`为`function`时：1.返回值不为`Promise`，调用`onRejectedNext(返回值)`

​																	2.返回值为`Promise`,调用`返回值.then(onRejectedNext)`

[JS错误处理机制](https://javascript.ruanyifeng.com/grammar/error.html)

### **4.new Promise中resolve传递的值为Promise的实例 **

**`resolve(new Promise)`**

`第一个Promise`中状态取决于`resolve中的Promise`

也就是说`resolve中的Promise`的then回调执行时就能确定`第一个Promise`的状态

例如下面这种使用形式：

```js
new Promise(function (resolve,reject) {
   return resolve(new Promise(function () {
       
   }))
}).then(function () {

},function () {

})
```

所以在实现`MyPromise`中的`_resolve`时，如果`_resolve(val)`中的值为Promise的实例,`instanceof val=== MyPromise`  那么在`val.then()`的两个回调中改变`MyPromise`的状态

## 代码

```js
function isFunction(fn) {
    return typeof fn === 'function'
}
const PENDING = 'PENDING'//进行中
const FULFILLED = 'FULFILLED'//已成功
const REJECTED = 'REJECTED' //已失败

class MyPromise{
    constructor(handle){
        this._value = null
        this._status = PENDING
        // 添加成功回调函数队列
        this._fulfilledQueues = []
        // 添加失败回调函数队列
        this._rejectedQueues = []
        try{
            handle(this._resolve.bind(this),this._reject.bind(this))
        }catch (err) {
            this._reject(err)
        }

    }
    _resolve(val){

        const run=() => {
            if (this._status !== PENDING) return//状态不可逆，只能从PENDING——》FULFILLED，不能从别的状态到FULFILLED
            // 依次执行成功队列中的函数，并清空队列
            const runFulfilled = (value) => {
                let cb;
                while (cb = this._fulfilledQueues.shift()) {
                    cb(value)
                }
            }

            // 依次执行失败队列中的函数，并清空队列
            const runRejected = (error) => {
                let cb;
                while (cb = this._rejectedQueues.shift()) {
                    cb(error)
                }
            }

            if(val instanceof MyPromise){
                val.then((value)=>{
                    this._value=value
                    this._status=FULFILLED
                    runFulfilled(value)
                },(err)=>{
                    this._value=err
                    this._status=REJECTED
                    runRejected(err)
                })
            }else {
                this._value=val
                this._status=FULFILLED
                runFulfilled(val)
            }
        }
        setTimeout(run,0)

    }

    _reject(err){
        if (this._status !== PENDING) return//状态不可逆，只能从PENDING——》REJECTED，不能从别的状态到REJECTED

        this._value= err
        this._status=REJECTED
        let cb
        while (cb=this._rejectedQueues.shift()){//依次执行失败队列中的函数，并清空队列
            cb(err)
        }
    }
}

MyPromise.prototype.then=function (onFulfilled,onRejected) {
    return new MyPromise( (onFulfilledNext,onRejectedNext) => {
        let fulfilled=(value) => {
            if(!isFunction(onFulfilled)) {
                onFulfilledNext(value) //值穿透
                return
            }

            try {
                let result = onFulfilled(value)
                if(result instanceof MyPromise){
                    result.then(onFulfilledNext,onRejectedNext)
                }else {
                    onFulfilledNext(result)
                }
            }catch (err) {
                onRejectedNext(err)
            }
        }

        let rejected=(value) => {
            if(!isFunction(onFulfilled)) {
                onRejectedNext(value)
                return
            }
            try{
                let result =onRejected(value)
                if(result instanceof  MyPromise){
                    result.then(onFulfilledNext,onRejectedNext)
                } else {
                    onRejectedNext(result)
                }
            }catch (err) {
                onRejectedNext(err)
            }
        }
        switch(this._status){
            case PENDING : this._fulfilledQueues.push(fulfilled)
                           this._rejectedQueues.push(rejected)
                           break
            case FULFILLED :fulfilled(this._value)
                            break
            case REJECTED : rejected(this._value)
                            break
        }
    })
}

//使用例子
new MyPromise(function (resolve,reject) {
    setTimeout(function () {
        resolve(new MyPromise(function (resolve) {
            resolve('promise in resolve')
        }))
    },1000)

}).then(function (val) {
    console.log('first then,get message :'+val)//1秒之后输出 first then,get message :promise in resolve
    return new MyPromise(function (resolve) {
        setTimeout(()=>{
            return resolve('promise in then')
        },1000)

    })
}).then(function (val) {
    console.log('second then ,get message:'+val)//2秒之后输出 second then ,get message:promise in then
})

```

