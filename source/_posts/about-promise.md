---
title: Promise原理小记
date: 2019-02-26 11:56:39
tags: 
- JavaScript
- Promise
categories: 前端开发
toc: true
---

# 定义

Promise 对象用于表示一个异步操作的最终状态（完成或失败），以及其返回的值。

***Promise***：为构造函数，接受一个函数作为参数，该函数接收resolve和reject两个参数，由JavaScript引擎提供，无需自己实现。

***resolve***：其作用是将Promise对象的状态从Pending变为Resolved,在异步操作成功时调用，并将异步操作的结果，作为参数传递出去；

***reject***：其作用是将Promise对象的状态从从Pending变为Rejected,在异步操作失败时调用，并将异步操作抛出的错误，作为参数传递出去。

Promise对象只有三种状态:

1. 未完成：pending
1. 已完成：resolved(也称fulfilled)
1. 失败：rejected

<!-- more -->

以上三种状态只有以下两种变化,一旦当前状态变为“resolved”或“rejected”，就意味着不会再有新的状态变化：

- pending->resolved
- pending->rejected

***then(onFulfilled, onRjected)***：then方法传递两个函数作为参数，当Promise状态为fulfilled时，调用 then 的 onFulfilled 方法，当Promise状态为rejected时，调用 then 的 onRjected 方法。

***catch*(onRjected)**：当Promise状态为rejected时，调用 then 的 onRjected 方法。
例子：

```js
let p = new Promise(function(resolve, reject){
    setTimeout(function(){
        let num = Math.random()
        if (num > 0.5) {
            resolve(num)
        }else{
            reject(num)
        }
    }, 1000)
})
p.then(function(num){
    console.log('大于0.5的数字：', num)
},function(num){
    console.log('小于等于0.5的数字', num)
})
// 运行第一次：小于等于0.5的数字 0.166162996031475
// 运行第二次：大于0.5的数字： 0.6591451548308984
```

# 原理

根据以上内容，Promise对象作为一个构造函数，并接受一个函数作为参数，该函数执行完同步或异步操作后，调用它的两个参数resolve和reject，，同时其内部有状态机制，原型链上有then方法，故可先初步实现Promise的功能，如下：

```js
function Promise(executor){ //executor是一个执行器（函数）
    let _this = this // 先缓存this以免后面指针混乱
    _this.status = 'pending' // 默认状态为等待态
    _this.value = undefined // 成功时要传递给成功回调的数据，默认undefined
    _this.reason = undefined // 失败时要传递给失败回调的原因，默认undefined
    function resolve(value) { // 内置一个resolve方法，接收成功状态数据
        // 上面说了，只有pending可以转为其他状态，所以这里要判断一下
        if (_this.status === 'pending') {
            _this.status = 'resolved' // 当调用resolve时要将状态改为成功态
            _this.value = value // 保存成功时传进来的数据
        }
    }
    function reject(reason) { // 内置一个reject方法，失败状态时接收原因
        if (_this.status === 'pending') { // 和resolve同理
            _this.status = 'rejected' // 转为失败态
            _this.reason = reason // 保存失败原因
        }
    }
    executor(resolve, reject) // 执行执行器函数，并将两个方法传入
}
// then方法接收两个参数，分别是成功和失败的回调，这里我们命名为onFulfilled和onRjected
Promise.prototype.then = function(onFulfilled, onRjected){
    let _this = this;   // 依然缓存this
    if(_this.status === 'resolved'){  // 判断当前Promise的状态
        onFulfilled(_this.value)  // 如果是成功态，当然是要执行用户传递的成功回调，并把数据传进去
    }
    if(_this.status === 'rejected'){ // 同理
        onRjected(_this.reason)
    }
}
```

测试下：

```js
// 成功
let Promise = require('./myPromise')  // 引入模块
let p = new Promise(function(resolve, reject){
  resolve('test')
})
p.then(function(data){
  console.log('成功', data)
},function(err){
  console.log('失败', err)
})
// 成功 test

// 失败
let Promise = require('./myPromise')  // 引入模块
let p = new Promise(function(resolve, reject){
  reject('test')
})
p.then(function(data){
  console.log('成功', data)
},function(err){
  console.log('失败', err)
})
// 失败 test
```

以上可以看到Promise传入参数的函数是立刻执行的，当传入的函数执行异步操作时，这个时候then方法中回调是不会异步执行的，即此时获取不到任何结果，例如：

```js
let p = new Promise(function(resolve, reject){
  setTimeout(function(){
    resolve(100)  
  }, 1000)
})
p.then(function(data){
  console.log('成功', data)
},function(err){
  console.log('失败', err)
})
// 不会输出任何代码
```

原因是then只是对成功或者失败状态进行了判断，而Promise实例化时，传入的函数会立即执行，而setTimeout是异步执行的，当then方法执行的时候，此时Promise的状态还是pendding，因此也需要对pendding状态进行判断，此时我们可以在then方法中判断pendding状态，先获取保存回调函数，等到异步函数执行再一一执行then方法中的回调函数，改进后的代码如下：

* 实现异步

```js
// myPromise
function Promise(executor){ //executor是一个执行器（函数）
    let _this = this // 先缓存this以免后面指针混乱
    _this.status = 'pending' // 默认状态为等待态
    _this.value = undefined // 成功时要传递给成功回调的数据，默认undefined
    _this.reason = undefined // 失败时要传递给失败回调的原因，默认undefined

    /**add: 用来存放then回调函数**/
    _this.onResolvedCallbacks = []; // 存放then成功的回调
    _this.onRejectedCallbacks = []; // 存放then失败的回调
    /**add**/

    function resolve(value) { // 内置一个resolve方法，接收成功状态数据
        // 上面说了，只有pending可以转为其他状态，所以这里要判断一下
        if (_this.status === 'pending') { 
            _this.status = 'resolved' // 当调用resolve时要将状态改为成功态
            _this.value = value // 保存成功时传进来的数据
            /**add**/
            _this.onResolvedCallbacks.forEach(function(fn){// 当成功的函数被调用时，之前缓存的回调函数会被一一调用
                fn()
            })
        }
    }
    function reject(reason) { // 内置一个reject方法，失败状态时接收原因
        if (_this.status === 'pending') { // 和resolve同理
            _this.status = 'rejected' // 转为失败态
            _this.reason = reason // 保存失败原因
            /**add**/
            _this.onRejectedCallbacks.forEach(function(fn){// 当失败的函数被调用时，之前缓存的回调函数会被一一调用
                fn()
            })
        }
    }
    executor(resolve, reject) // 执行执行器函数，并将两个方法传入
}
// then方法接收两个参数，分别是成功和失败的回调，这里我们命名为onFulfilled和onRjected
Promise.prototype.then = function(onFulfilled, onRjected){
    let _this = this;   // 依然缓存this
    /**add: 判断pendding状态，将成功或者失败的回调函数存放到对应的数组中**/
    if(_this.status === 'pending'){
        _this.onResolvedCallbacks.push(function(){
            onFulfilled(_this.value)
        })
        _this.onRejectedCallbacks.push(function(){
            onRjected(_this.reason)
        })
    }
    /**add**/
    if(_this.status === 'resolved'){  // 判断当前Promise的状态
        onFulfilled(_this.value)  // 如果是成功态，当然是要执行用户传递的成功回调，并把数据传进去
    }
    if(_this.status === 'rejected'){ // 同理
        onRjected(_this.reason)
    }
}
module.exports = Promise  // 导出模块，否则别的文件没法使用

```

* 处理错误

Promise捕获错误执行的是reject函数，因此在实例中跑出错误时，只执行reject函数就可以，可使用try catch进行异常捕获；

```js
try{
    executor(resolve, reject)
} catch(e){ // 如果捕获发生异常，直接调失败，并把参数穿进去
    reject(e)
}

```

* catch方法实现

catch方法其实就是获取实例rejected方法捕获错误，相当于执行then(null, onRejected)

```js
Promise.prototype.catch = function(onRejected) {
  return this.then(null, onRejected)
}
```

* 实现then的链式调用

上面实现的then方法其实还不完善，不能进行链式调用。是否可以跟jq一样在then方法中return this实现链式调用，可实际上Promise中then方法要复杂的多，不能简单的用return this实现链式调用。

```js
let p1 = new Promise(function(resolve, reject){
  resolve()
})
let p2 = p1.then(function(data){ //这是p1的成功回调，此时p1是成功态
    throw new Error('错误') // 如果这里抛出错误，p2应是失败态
})
p2.then(function(){

},function(err){
    console.log(err)
})
// Error: 错误

```

按照上面代码，如果then方法中返回的是this，那么p2跟p1相同，固状态也相同，但是Promise的成功态和失败态不能相互转换，那就不会得到p1成功而p2失败的效果，而实际上是可能发生这种情况的。因此Promise的then方法实现链式调用的原理是：**返回一个新的Promise**

按照这样的思路，在then方法中先定义一个新的Promise，取名为promise2，然后在三种状态下分别用promise2包装一下，在调用onFulfilled时用一个变量x（规定的）接收返回值，try...catch...一下代码，没错就调resolve传入x，有错就调reject传入错误，最后再把promise2给return出去。

```js
// 改动then
let promise2;
if (_this.status === 'resolved') {
    promise2 = new Promise(function (resolve, reject) {
        try {
            let x = onFulfilled(_this.value)
            resolve(x)
        } catch (e) {
            reject(e)
        }
    })
}
if (_this.status === 'rejected') {
    promise2 = new Promise(function (resolve, reject) {
        try {
            let x = onRjected(_this.reason)
            resolve(x)
        } catch (e) {
            reject(e)
        }
    })
}
if(_this.status === 'pending'){
    promise2 = new Promise(function (resolve, reject) {
        _this.onResolvedCallbacks.push(function(){
            try {
                let x = onFulfilled(_this.value)
                resolve(x)
            } catch (e) {
                reject(e)
            }
        })
        _this.onRejectedCallbacks.push(function(){
            try {
                let x = onRjected(_this.reason)
                resolve(x)
            } catch (e) {
                reject(e)
            }
        })
    })
}
return promise2
```

这样实现x只是当做一个普通的值，比如字符串数字数组对象等值可以用上面代码简单链式调用传给下一个then，但是如果x也是一个Promise或者是其他异步操作，可能性如下：

1. 前一次then返回一个普通值，比如字符串数组对象等，只需传给下一个then，刚才的方法就能实现。
2. 前一次then返回的是一个Promise，是正常的操作，也是Promise提供的语法糖，我们要想办法判断到底返回的是啥。
3. 前一次then返回的是一个Promise，其中有异步操作，也是理所当然的，那我们就要等待他的状态改变，再进行下面的处理。
4. 前一次then返回的是自己本身这个Promise

```js
var p1 = p.then(function(){
  return p1  
})
```

5. 前一次then返回的是自定义的Promise的普通对象，比如{then:'xxx'}，或者改变then方法的访问器属性，在then里故意抛错。比如

```js
let promise = {}
Object.defineProperty(promise,'then',{
    value: function(){
        throw new Error('报错气死你')
    }
})
```

6. 在resolve传一个Promise对象

```js
p.then(function(data) {
    return new Promise(function(resolve, reject) {
      resolve(new Promise(function(resolve,reject){
        resolve(1111)
      }))
    })
})
```

7. 同时调用resolve和reject，则需取前面调用的方法，忽略后一个
8. 不传任何参数，下面代码运行结果是undefined，按我们上面实现的代码，then不传任何参数时，resolve中传的值是没有办法穿透的，

```js
new Promise(resolve=>resolve(8))
  .then()
  .then()
  .then(function foo(value) {
    alert(value)
  })
```

针对以上问题可以对then方法代码做出优化：

```js
Promise.prototype.then = function (onFulfilled, onRjected) {
    //成功和失败默认不传给一个函数，解决了问题8
    onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : function (value) {
        return value;
    }
    onRjected = typeof onRjected === 'function' ? onRjected : function (err) {
        throw err;
    }
    let _this = this;
    let promise2; //返回的promise
    if (_this.status === 'resolved') {
        promise2 = new Promise(function (resolve, reject) {
            // 当成功或者失败执行时有异常那么返回的promise应该处于失败状态
            setTimeout(function () {// 根据规范让那俩家伙异步执行
                try {
                    let x = onFulfilled(_this.value);//这里解释过了
                    // 写一个方法统一处理问题1-7
                    resolvePromise(promise2, x, resolve, reject);
                } catch (e) {
                    reject(e);
                }
            })
        })
    }
    if (_this.status === 'rejected') {
        promise2 = new Promise(function (resolve, reject) {
            setTimeout(function () {
                try {
                    let x = onRjected(_this.reason);
                    resolvePromise(promise2, x, resolve, reject);
                } catch (e) {
                    reject(e);
                }
            })
        })
    }
    if (_this.status === 'pending') {
        promise2 = new Promise(function (resolve, reject) {
            _this.onResolvedCallbacks.push(function () {
                setTimeout(function () {
                    try {
                        let x = onFulfilled(_this.value);
                        resolvePromise(promise2, x, resolve, reject);
                    } catch (e) {
                        reject(e)
                    }
                })
            });
            _this.onRejectedCallbacks.push(function () {
                setTimeout(function () {
                    try {
                        let x = onRjected(_this.reason);
                        resolvePromise(promise2, x, resolve, reject);
                    } catch (e) {
                        reject(e);
                    }
                })
            });
        })
    }
    return promise2;
}

function resolvePromise(promise2, x, resolve, reject) {
    // 接受四个参数，新Promise、返回值，成功和失败的回调
    if (promise2 === x) { //这里应该报一个类型错误，来解决问题4
        return reject(new TypeError('循环引用了'))
    }
    // 若返回一个promise实例
    if (x instanceof Promise) {
        // x状态没确定时即实例化时进行异步操作
        if (x.status === 'pending') {
            x.then(function(v) {
                resolvePromise(promise2, v, resolve, reject)
            }, reject)
        } else {
            // 状态确定时，直接调用then方法
            x.then(resolve, reject)
        }
        return
    }
    // 看x是不是一个promise,promise应该是一个对象
    let called; // 表示是否调用过成功或者失败，用来解决问题7
    //下面判断上一次then返回的是普通值还是函数，来解决问题1、2
    if (x !== null && (typeof x === 'object' || typeof x === 'function')) {
        // 可能是promise {},看这个对象中是否有then方法，如果有then我就认为他是promise了
        try {
            let then = x.then;// 保存一下x的then方法
            if (typeof then === 'function') {
                // 成功
                //这里的y也是官方规范，如果还是promise，可以当下一次的x使用
                //用call方法修改指针为x，否则this指向window
                then.call(x, function (y) {
                    if (called) return //如果调用过就return掉
                    called = true
                    // y可能还是一个promise，在去解析直到返回的是一个普通值
                    resolvePromise(promise2, y, resolve, reject)//递归调用，解决了问题6
                }, function (err) { //失败
                    if (called) return
                    called = true
                    reject(err);
                })
            } else {
                resolve(x)
            }
        } catch (e) {
            if (called) return
            called = true;
            reject(e);
        }
    } else { // 说明是一个普通值1
        resolve(x); // 表示成功了
    }
}
```

* 其他方法

```js
    // 捕获错误的方法，在原型上有catch方法，返回一个没有resolve的then结果即可
    Promise.prototype.catch = function (callback) {
        return this.then(null, callback)
    }
    // 解析全部方法，接收一个Promise数组promises,返回新的Promise，遍历数组，都完成再resolve
    Promise.all = function (promises) {
        //promises是一个promise的数组
        return new Promise(function (resolve, reject) {
            let arr = []; //arr是最终返回值的结果
            let i = 0; // 表示成功了多少次
            function processData(index, y) {
                arr[index] = y;
                if (++i === promises.length) {
                    resolve(arr);
                }
            }
            for (let i = 0; i < promises.length; i++) {
                promises[i].then(function (y) {
                    processData(i, y)
                }, reject)
            }
        })
    }
    // 只要有一个promise成功了 就算成功。如果第一个失败了就失败了
    Promise.race = function (promises) {
        return new Promise(function (resolve, reject) {
            for (var i = 0; i < promises.length; i++) {
                promises[i].then(resolve,reject)
            }
        })
    }
    // 生成一个成功的promise
    Promise.resolve = function(value){
        return new Promise(function(resolve,reject){
            resolve(value);
        })
    }
    // 生成一个失败的promise
    Promise.reject = function(reason){
        return new Promise(function(resolve,reject){
            reject(reason);
        })
    }
```