---
title: Async原理小记
date: 2019-02-26 11:12:31
tags: 
- JavaScript
- Promise
categories: 前端开发
toc: true
---

## 定义

**async function** 声明用于定义一个返回 **AsyncFunction** 对象的异步函数。异步函数是指通过事件循环异步执行的函数，它会通过一个隐式的 **Promise** 返回其结果。但是如果你的代码使用了异步函数，它的语法和结构会更像是标准的同步函数。

```js
function resolveAfter2Seconds() {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve('resolved');
    }, 2000);
  });
}

async function asyncCall() {
  console.log('calling');
  var result = await resolveAfter2Seconds();
  console.log(result);
  // expected output: 'resolved'
}

asyncCall();

// calling
// resolved
```

<!-- more -->

## 描述

当调用一个 **async** 函数时，会返回一个 **Promise** 对象，可以使用then方法添加回调函数。 **async** 函数返回一个值时，**Promise** 的 **resolve** 方法会负责传递这个值；当 **async** 函数抛出异常时，**Promise** 的 **reject** 方法也会传递这个异常值。

当 **async** 函数执行的时候，一旦遇到await就会先返回，等到异步操作完成，再接着执行函数体内后面的语句。

```js
// then写法
function readFile(a){
    return new Promise(resolve=>{
          setTimeout(()=>{
              console.log(a);
              resolve(a);
          },500)
    })
}

readFile('a')
.then(()=>{
   console.log('b')
})
.then(()=>{
    console.log('c')
});     //a b c

// async写法
async function test(){
  await readFile('a');
  console.log('b');
  console.log('c');
}
test();   //a b c
```

> 注意：await 关键字仅仅在 async function中有效，并且await只会影响async函数内部的执行顺序。如果在 async function函数体外使用 await ，你只会得到一个语法错误（SyntaxError）。

## 实现原理

理解async原理之前先了解Generator函数，其实async是Generator的语法糖，async 就等于Generator+自动执行器。

### Generator函数

Generator是 ES6 提供的一种异步编程解决方案，返回一个遍历器对象。它与普通函数的区别是：

1. function关键字与函数名之间有一个星号；
2. 函数体内部使用yield语句，定义不同的内部状态。

```js
function* g() {
    yield 'a';
    yield 'b';
    yield 'c';
    return 'ending';
}

g(); //返回一个对象
```

上面例子g()不会执行，而是会返回一个迭代器对象（Iterator Object）。

```js
var gen = g();
gen.next(); // 返回Object {value: "a", done: false}
```

该对象有一个next方法，调用该方法时会进行分段执行。
gen.next()返回一个对象{value: "a", done: false}，'a'就是g函数执行到第一个yield语句之后得到的值，false表示g函数还没有执行完，只是在这暂停。以此类推，当执行到第四个next()时，返回{value: "ending", done: true}，这样，整个g函数就运行完毕了。

综上：调用 Generator 函数，返回一个遍历器对象，代表 Generator 函数的内部指针。以后，每次调用遍历器对象的next方法，就会返回一个有着value和done两个属性的对象。value属性表示当前的内部状态的值，是yield表达式后面那个表达式的值；done属性是一个布尔值，表示是否遍历结束。

### yield 表达式

由于 Generator 函数返回的遍历器对象，只有调用next方法才会遍历下一个内部状态，所以其实提供了一种可以暂停执行的函数。yield表达式就是暂停标志。

遍历器对象的next方法的运行逻辑如下。

（1）遇到yield表达式，就暂停执行后面的操作，并将紧跟在yield后面的那个表达式的值，作为返回的对象的value属性值。

（2）下一次调用next方法时，再继续往下执行，直到遇到下一个yield表达式。

（3）如果没有再遇到新的yield表达式，就一直运行到函数结束，直到return语句为止，并将return语句后面的表达式的值，作为返回的对象的value属性值。

（4）如果该函数没有return语句，则返回的对象的value属性值为undefined。

Generator 函数可以不用yield表达式，这时就变成了一个单纯的暂缓执行函数。

```js
function* f() {
  console.log('执行了！')
}

var generator = f();

setTimeout(function () {
  generator.next()
}, 2000);
```

上面代码中，函数f如果是普通函数，在为变量generator赋值时就会执行。但是，函数f是一个 Generator 函数，就变成只有调用next方法时，函数f才会执行。

> 注意：yield表达式只能用在 Generator 函数里面，用在其他地方都会报错。

现在回过头再来看，将开头那段代码改成Generator函数

```js
function readFile(a){
    return new Promise(resolve=>{
          setTimeout(()=>{
              console.log(a);
              resolve(a);
          },500)
    })
}
function *foo(){
  var result = yield readFile('a');
  console.log('b');
}
var it = foo();
var result = it.next();  //next返回的value是readFile函数返回的Promise对象
result.value.then(()=>{   //给Promise对象增加成功的回调
  it.next();   //当Promise成功后恢复foo()函数执行
});  //a b
```

使用Generator函数执行时，每次都需要next()来执行，可以看出控制foo函数的暂停和继续却需要写额外的代码，如何控制Generator函数自动执行next() 方法？针对上面代码可以用递归来实现。

```js
function run(g){
  var res = g.next(); //res.value是个promise对象
  if(!res.done){
    res.value.then(()=>{   //promise解决了才调用next()继续执行生成器内部函数
      run(g);
    })  
  }
}
// test
function readFile(a){
    return new Promise(resolve=>{
          setTimeout(()=>{
              console.log(a);
              resolve(a);
          },500)
    })
}
function *foo(){
  console.log('a');
  var result = yield readFile('b');
  console.log('c');
}

run(foo());
console.log('d');
foo()
```

运行结果为a d b c。分析：在js运行中，第一次执行foo().next()会打印出a，然后生成器内函数暂停，因为readFile返回一个promise对象并且进入setTimeout进行异步执行，此时进入异步队列等待执行，主线程继续运行，再打出d，等定时器到了，异步队列依次执行，打出b等到结果后，再执行yield后面的代码，打出c。再看async写法：

```js
// async写法
async function foo() {
  console.log('a');
  var result = await readFile('b');
  console.log('c');
}
```

可以看出async函数写法，Generator函数函数很相近，上面代码只是简单模拟async的原理，async可以看成Generator+自动执行器的语法糖。