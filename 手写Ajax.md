# 手写Ajax（jsonp）

参考：[原生 JavaScript 实现 AJAX、JSONP](https://juejin.im/entry/589921640ce46300560ef894)

## XMLHttpRequest的使用

参考：[XMLHttpRequest](https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest)

​			[XMLHttpRequest发送请求和获取响应](https://blog.csdn.net/virusos/article/details/71272104)

例子：

```js
var request = new XMLHttpRequest();
request.open("GET", "get.php", true);
request.send();
//该属性每次变化时会触发
request.onreadystatechange = function(){
    //若响应完成且请求成功
    if(request.readyState === 4 && request.status === 200){
        //do something, e.g. request.responseText
    }
}
```

**发送请求：**

1. 建立XMLHttpRequest对象：var request = new XMLHttpRequest();
2. .open初始化请求：request.open(方法名, URL, 是否异步);
3. .send(data)发送数据：request.send(data);

**监听回复：**

​	1.服务器回复之后`readystate`变化，调用`onreadystatechange`

​	2.根据`readystate`请求状态码，`status`服务器返回的响应状态。处理请求成功之后服务器返回的数据

## 不考虑json的ajax

**ajax的使用方式：**

```js
ajax({ 
 url: 'test.php',  // 请求地址
 type: 'POST',  // 请求类型，默认"GET"，还可以是"POST"
 data: {'b': '异步请求'},  // 传输数据
 success: function(res){  // 请求成功的回调函数
  console.log(JSON.parse(res)); 
 },
 error: function(error) {}  // 请求失败的回调函数
});
```



**需求分析：**

​				根据`type`的不同发送不同请求，根据服务器返回的状态调用`susccess`请求成功，`error`请求失败函数

```js
function  ajax(params) {
    let {method='GET',data={},url,success=function () {},error=function () {}}=params
    let request = new XMLHttpRequest()
    request.open(method,url,true)
    request.send(data)
    request.onreadystatechange=function () {
        if(request.readyState === 4){
            let status = request.status
            if(request.status>=200&&request.status<300){
                success(request.responseText,request.responseXML)
            }else {
                error(status)
            }
        }
    }
}
```

**功能完善：**

​				1.为保证服务器接收数据正确，如果数据类型为`key1=val1&key2=val2`时客户端所发送的数据都要经`encodeURIComponent`进行编码。

​							也就是：1.1发送`GET`请求

​											1.2发送`POST`请求的`Content-Type='application/x-www-form-urlencoded'`时

​				2.`GET`方法`send(null)`,POST方法根据`Content-Type`的不同发送不同的数据数据类型

​					本例中的`Content-Type`可为：`application/json`、`application/x-www-form-urlencoded`

```js
let TYPE_URLENCODED='application/x-www-form-urlencoded'
let TYPE_JSON = 'application/json'
function  ajax(params) {
    let method = params.method.toUpperCase()||'GET'
    let data = params.data||{}
    let url = params.url
    let success = params.success||function () {}
    let error =params.error||function () {}
    let contentType = params.contentType||'application/x-www-form-urlencoded'
    if(!url){
        console.log('url can\'t be undefined')
        return
    }

    const request = new XMLHttpRequest()

    function urlEncodeFormat(data){
        let encoded=[]
        for (let key in data){
           let unit=encodeURIComponent(key)+'='+encodeURIComponent(data[key])
               encoded.push(unit)
        }
        return encoded.join('&')
    }

    if(method==='GET'){
        data=urlEncodeFormat(data)
        debugger
        let url=params.url+'?'+data
        request.open(method,url,true)
        request.send(null)

    }else if(method==='POST'){
        request.open('POST',url,true)
        if(contentType === TYPE_URLENCODED){
            data=urlEncodeFormat(data)
        }else if( contentType === TYPE_JSON){
            data=JSON.stringify(data)
        }
        request.setRequestHeader('Content-Type',contentType)
        request.send(data)
    }
    request.onreadystatechange = function () {
        if (request.readyState === 4){
            if(request.status>=200&& request.status<300){
                success(request.responseText,request.responseXML)
            }else{
                error(request.status)
            }
        }
    }
}

//使用示例
let baseURL='http://localhost:3333'
ajax({
    url:baseURL+'/getTest',
    method:'GET',
    data:{
        name:'get data',
        data:'gggg'
    },
    success:function (data) {
        console.log(data)
    },
    error:function (err) {
        console.log(err)
    }
})

ajax({
    url:baseURL+'/postTest',
    method:'POST',
    data:{
        name:'post data',
        data:'ppppppp'
    },
    success:function (data) {
        console.log(data)
    },
    error:function (err) {
        console.log(err)
    }
})

ajax({
    url:baseURL+'/postTest',
    method:'POST',
    contentType:'application/json',
    data:{
        name:'post data',
        data:'jjjjj'
    },
    success:function (data) {
        console.log(data)
    },
    error:function (err) {
        console.log(err)
    }
})

```

## JSONP

**JSONP原理：**1.`<script>`标签不受同源政策影响，可以跨域根据`scr`的地址请求资源

​						2.1用`<sctipt>`标签请求到的是`javascript`代码，相当于浏览器直接请求到了这段代码并且执行。服务器向客户端jsonp传参就是在返回的javasctipt代码的函数中传参，类似`jsonpCallBack({data:'ddddd'})`。

​						2.2  succuss:    `jsonpCallBack`在发送请求前被声明，会拿到服务器返回的参数，调用`success(data)`,将参数传给`success`

​						2.3  error：由于没有使用`XMLHttpRequest`，所以无法根据服务器返回状态调用`error`——>设置超时定时器，如果一段时间内没有调用`jsonpCallBack`成功调用`success`说明超时、请求错误



**仅仅实现了客户端，服务端PHP请参考：**[原生 JavaScript 实现 AJAX、JSONP](https://juejin.im/entry/589921640ce46300560ef894)

```js
let TYPE_URLENCODED='application/x-www-form-urlencoded'
let TYPE_JSON = 'application/json'
function  ajax(params) {
    let data = params.data||{}
    let url = params.url
    let success = params.success||function () {}
    let error =params.error||function () {}
    let jsonpCbCount=0
    if(!url){
        console.log('url can\'t be undefined')
        return
    }

    function urlEncodeFormat(data){
        let encoded=[]
        for (let key in data){
            let unit=encodeURIComponent(key)+'='+encodeURIComponent(data[key])
            encoded.push(unit)
        }
        return encoded.join('&')
    }

    function normalRequest() {
        let method = params.method.toUpperCase()||'GET'
        let contentType = params.contentType||'application/x-www-form-urlencoded'
        const request = new XMLHttpRequest()

        if(method==='GET'){
            data=urlEncodeFormat(data)
            let url=params.url+'?'+data
            request.open(method,url,true)
            request.send(null)

        }else if(method==='POST'){
            request.open('POST',url,true)
            if(contentType === TYPE_URLENCODED){
                data=urlEncodeFormat(data)
            }else if( contentType === TYPE_JSON){
                data=JSON.stringify(data)
            }
            request.setRequestHeader('Content-Type',contentType)
            request.send(data)
        }
        request.onreadystatechange = function () {
            if (request.readyState === 4){
                if(request.status>=200&& request.status<300){
                    success(request.responseText,request.responseXML)
                }else{
                    error(request.status)
                }
            }
        }
    }

    function jsonpRequest() {
        let method = 'GET'
        let timeout=params.timeout||1000
        let data = params.data||{}
        let jsonpCbFn='jsonpCb'+jsonpCbCount//每次jsonp请求返回的script调用的函数都是唯一的
        jsonpCbCount++

        //添加script发送请求
        data.jsonpCbFn=jsonpCbFn
        data= urlEncodeFormat(data)
        let script = document.createElement('script')
        script.src=params.url+'?'+data
        let head=document.getElementsByTagName('head')[0]
        head.appendChild(script)

        //定义服务器返回数据时调用的函数
        window[jsonpCbFn] = function (res) {
            clearTimeout(timer)
            head.removeChild(script)
            success(res)
        }

        let timer = setTimeout(function () {
            head.removeChild(script)
            params.error('jsonp error,timeout:'+timeout)
        },timeout)
    }

    params.jsonp?jsonpRequest():normalRequest()

}

//使用示例
let baseURL='http://localhost:3333'
ajax({
    url:baseURL+'/jsonpTest',
    jsonp:true,
    data:{
        name:'jsonp',
        data:'jsonp/jsonp/jsonp'
    },
    success:function (data) {
        console.log(data)
    },
    error:function (err) {
        console.log(err)
    }
})

```



​	