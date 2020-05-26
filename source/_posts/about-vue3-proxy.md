---
title: 浅析Vue3数据响应系统
date: 2019-10-11 20:08:23
tags: 
- JavaScript
- Vue
categories: 前端开发
toc: true
---

国庆长假还没结束，尤大大就把Vue3源码给放了出来，还能不能好好让人好好放个假😨，目前的版本还是 <code>Pre-Alpha</code>，只是大概完成了核心的功能，还不太完整，源码地址：[Vue-next](https://github.com/vuejs/vue-next)， 而正式版本可能要明年才发布了，于是放假回来，便第一时间clone了源码，生啃了起来，还别说，真香~~~😂  

源码整体结构还是比较清晰，相比较于Vue2.x做了比较大的调整，代码话说98%都是用ts编写，所以看源码还需要大概了解ts的一些知识。而在还没发布Vue3源码之前，官方也已经给出了 [Vue Composition API RFC](https://vue-composition-api-rfc.netlify.com/#basic-example)，可以初步了解Vue3的一些写法和特性，这两天花了些时间大概看了下数据响应部分（[网上已经有大佬给出了调试技巧](https://juejin.im/post/5d99d9a0f265da5b8601264c)）

<!-- more -->

熟悉 <code>Vue2.x</code> 的童鞋应该知道，其数据响应是基于 <code>ES5</code> 的API <code>Object.defineProperty</code> 来操作属性的 <code>getter/setter</code> 实现的，本身API存在一些不足：比如对于数组，不能通过数组下标来观测其变化，也不能观测数组长度length的变化，为此对数组7个原生方法做了拦截处理实现对数组的观测。对于对象，由于defineProperty的局限性，<code>Vue2.x</code> 不能检测对象属性的添加或删除。

为了优化解决这些不足，更好的实现方式是通过 <code>ES6</code> 提供的 <code>Proxy</code> 对象。

<code>Vue3</code> 就是基于 <code>Proxy</code> 对其数据响应系统进行了重写，现在这部分可以作为独立的模块配合其他框架使用。数据响应可分为三个阶段：**初始化阶段 --> 依赖收集阶段 --> 数据响应阶段**

## Proxy代理须知

用 <code>Proxy</code> 做代理时，我们需要了解几个问题：

1、<code>Proxy</code> 代理是如何对其 <code>trap</code> 进行处理来实现数据响应的？也就是其 <code>get/set</code> 里面是如何做拦截处理（其实这里的trap默认行为可以通过 <code>Reflect</code> 来返回，<code>Reflect</code> 对象的方法与 <code>Proxy</code> 对象的方法一一对应，只要是 <code>Proxy</code> 对象的方法，就能在 <code>Reflect</code> 对象上找到对应的方法。这就让 <code>Proxy</code> 对象可以方便地调用对应的 <code>Reflect</code> 方法，完成默认行为，作为修改行为的基础。这里具体可以查看阮大神的[ES6入门](http://es6.ruanyifeng.com/?search=iteration&x=0&y=0#docs/reflect)）

2、<code>Proxy</code> 代理的对象只能代理到第一层，当代理的对象多层嵌套时，那么对象内部的深度监测需要如何去实现？  

3、当代理对象是数组时，比如push操作会触发多次 <code>get/set</code>，因为push操作除了增加数组的数据项之外，也会引发数组本身其他相关属性的改变，因此会多次触发 <code>get/set</code> ，那么要如何解决呢？

4、<code>Proxy</code> 不能对基本类型数据进行代理，那Vue3是通过什么实现基本类型数据的响应的？在Vue3中，通ts自定义了一种特殊类型的数据<code>Ref</code>，对于此类型的理解需要有一定的ts基础，而基本类型数据就是通过它的转化变成了可响应式，具体如何实现后面我们再来分析，这里先不做解释了。

下面我们会稍微分析下 Vue3 针对这几个问题做了哪些优化处理。

## 初始化阶段

初始化过程相对比较简单，通过 <code>reactive()</code> 方法将数据转化成 <code>Proxy</code> 对象，这里注意一个比较重要的对象 <code>targetMap</code>，它在依赖收集阶段起着比较重要的作用，具体下面会有分析。

```js
export function reactive(target: object) {
  // if trying to observe a readonly proxy, return the readonly version.
  if (readonlyToRaw.has(target)) {
    return target
  }
  // target is explicitly marked as readonly by user
  if (readonlyValues.has(target)) {
    return readonly(target)
  }
  return createReactiveObject(
    target,
    rawToReactive,
    reactiveToRaw,
    mutableHandlers,
    mutableCollectionHandlers
  )
}

...

// 创建proxy对象
function createReactiveObject(
  target: any,
  toProxy: WeakMap<any, any>,
  toRaw: WeakMap<any, any>,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>
) {
  if (!isObject(target)) {
    if (__DEV__) {
      console.warn(`value cannot be made reactive: ${String(target)}`)
    }
    return target
  }
  // target already has corresponding Proxy
  let observed = toProxy.get(target)
  if (observed !== void 0) {
    return observed
  }
  // target is already a Proxy
  if (toRaw.has(target)) {
    return target
  }
  // only a whitelist of value types can be observed.
  if (!canObserve(target)) {
    return target
  }
  const handlers = collectionTypes.has(target.constructor)
    ? collectionHandlers
    : baseHandlers
  observed = new Proxy(target, handlers)
  toProxy.set(target, observed)
  toRaw.set(observed, target)
  if (!targetMap.has(target)) {
    targetMap.set(target, new Map())
  }
  return observed
}
```

Vue3 如何进行深度观测的？先看下面这段代码

```js
let data = { x: {y: {z: 1 } } }
let p = new Proxy(data, {
  get(target, key, receiver) {
    const res = Reflect.get(target, key, receiver)
    console.log('get value:', key)
    console.log(res)
    return res
  },
  set(target, key, value, receiver) {
    console.log('set value:', key, value)
    return Reflect.set(target, key, value, receiver)
  }
})
p.x.y = 2

// get value: x
// {y: 2}
```

上面代码我们可以知道 <code>Proxy</code> 只会代理一层，因为这里只是触发了一次最外层属性 <code>x</code> 的 <code>get</code>，而重新赋值的其内部属性 <code>y</code>，此时 <code>set</code> 并没有被触发，所以改变内部属性是不会监测到的。继续看，Reflect.get返回的结果正是 <code>target</code> 的内层结构，此时<code>p.x.y</code>的值也已经变成 <code>2</code> 了，我们可以判断当前 <code>Reflect.get</code> 返回的值是否为 <code>object</code>，若是则再通过 <code>reactive</code> 做代理，这样就达到了深度观测的目的了。

Vue3实现过程具体我们可以看下面源码：

```js
function createGetter(isReadonly: boolean) {
  return function get(target: any, key: string | symbol, receiver: any) {
    const res = Reflect.get(target, key, receiver)
    if (typeof key === 'symbol' && builtInSymbols.has(key)) {
      return res
    }
    if (isRef(res)) {
      return res.value
    }
    track(target, OperationTypes.GET, key)
    // 当代理的对象是多层结构时，Reflect.get会返回对象的内层结构，我们可以拿到当前res再做判断是否为object，进而进行reactive，就达到了深度观测的目的了
    return isObject(res)
      ? isReadonly
        ? // need to lazy access readonly and reactive here to avoid
          // circular dependency
          readonly(res)
        : reactive(res)
      : res
  }
}
```

## 依赖收集阶段

所谓的依赖在Vue3可简单理解为各种 <code>effect</code> 响应式函数，其中包括了属性依赖的 <code>effect</code>，计算属性 <code>computedEffect</code> 以及组件视图的 <code>componentEffect</code>

1、在视图挂载渲染时会执行一个 <code>componentEffect</code>，触发相关数据属性getter操作来完成视图依赖收集。

2、<code>effect</code> 函数执行也会触发相关属性的getter操作，此时操作了某个属性的 <code>effect</code> 也会被该属性对应进行收集（注意这里的属性是可观测的）。

之所以说是响应式的，是因为effect方法回调中关联了被观测的数据属性，而effect一般是立即执行的，此时触发了该属性的 <code>getter</code>，进行依赖收集，当该属性触发 <code>setter</code> 时，便会触发执行收集的依赖。另外，这里每次effect执行时，当前的effect会被压入一个名为 <code>activeReactiveEffectStack</code> 的栈中，是在依赖收集的时候使用。

```js
export function effect(
  fn: Function,
  options: ReactiveEffectOptions = EMPTY_OBJ
): ReactiveEffect {
  if ((fn as ReactiveEffect).isEffect) {
    fn = (fn as ReactiveEffect).raw
  }
  const effect = createReactiveEffect(fn, options)
  if (!options.lazy) {
    // effect立即执行，触发effect回调函数fn中相关响应数据属性的getter操作，从而进行依赖收集
    effect()
  }
  return effect
}
...
// 触发getter操作，进行依赖收集
export function track(
  target: any,
  type: OperationTypes,
  key?: string | symbol
) {
  if (!shouldTrack) {
    return
  }
  const effect = activeReactiveEffectStack[activeReactiveEffectStack.length - 1]
  if (effect) {
    if (type === OperationTypes.ITERATE) {
      key = ITERATE_KEY
    }
    let depsMap = targetMap.get(target)
    if (depsMap === void 0) {
      targetMap.set(target, (depsMap = new Map()))
    }
    let dep = depsMap.get(key!)
    if (dep === void 0) {
      depsMap.set(key!, (dep = new Set()))
    }
    // 防止依赖重复收集
    if (!dep.has(effect)) {
      dep.add(effect)
      effect.deps.push(dep)
      if (__DEV__ && effect.onTrack) {
        effect.onTrack({
          effect,
          target,
          type,
          key
        })
      }
    }
  }
}
```

开头说过 <code>targetMap</code> 对象在依赖收集过程中的重要作用，看源码我们大概知道了，它维护了一个依赖收集的关系表，<code>targetMap</code> 是一个 <code>WeakMap</code>，其 <code>key</code> 值是当前被代理的对象 <code>target</code>，而 <code>value</code> 则是该对象所对应的 <code>depsMap</code>，它是一个 <code>Map</code>，<code>key</code> 值为触发 <code>getter</code> 时的属性值，而 <code>value</code> 值则是触发过该属性值所对应的各个 <code>effect</code>。

故 <code>targetMap</code> 的关系映射可以看成 <code>target --> key --> effect</code>，可以看出 <code>target</code> 被观测后，其属性 <code>key</code> 在被触发 <code>getter</code> 操作时，收集了所依赖的 <code>effect</code>，可以说 <code>targetMap</code> 是Vue3进行依赖收集的一个核心对象。

## 响应阶段

当触发属性 <code>setter</code> 时，通过 <code>trigger</code> 函数会执行属性对应收集的 <code>effects</code>，也包括 <code>computedEffects</code>，此时通过 <code>scheduleRun</code> 逐个调用 <code>effect</code>，最后完成视图更新。

上面我们讲过监测数组的时候可能触发多次 <code>get/set</code>, 那么如何防止触发多次的呢？先看Vue3的源码(简写省略了部分代码)：

```js
// setter操作触发响应
function set(
  target: any,
  key: string | symbol,
  value: any,
  receiver: any
): boolean {
  value = toRaw(value)
  // 判断key是否为当前target自身属性
  const hadKey = hasOwn(target, key)
  // 获取旧值
  const oldValue = target[key]
  if (isRef(oldValue) && !isRef(value)) {
    oldValue.value = value
    return true
  }
  const result = Reflect.set(target, key, value, receiver)
  ...
  if (!hadKey) {
    // 若属性不存在标记为add操作
    trigger(target, OperationTypes.ADD, key)
  } else if (value !== oldValue) {
    // 若值不相等在触发，并且标记为set操作
    trigger(target, OperationTypes.SET, key)
  }
  ...
  return result
}

export function trigger(
  target: any,
  type: OperationTypes,
  key?: string | symbol,
  extraInfo?: any
) {
  const depsMap = targetMap.get(target)
  if (depsMap === void 0) {
    // never been tracked
    return
  }
  const effects = new Set<ReactiveEffect>()
  const computedRunners = new Set<ReactiveEffect>()
  // 这里遍历找出相关依赖的effect
  if (type === OperationTypes.CLEAR) {
    // collection being cleared, trigger all effects for target
    depsMap.forEach(dep => {
      addRunners(effects, computedRunners, dep)
    })
  } else {
    // schedule runs for SET | ADD | DELETE
    if (key !== void 0) {
      addRunners(effects, computedRunners, depsMap.get(key))
    }
    // also run for iteration key on ADD | DELETE
    if (type === OperationTypes.ADD || type === OperationTypes.DELETE) {
      // 这里当改变数组length长度时也会触发相关effect进行响应
      const iterationKey = Array.isArray(target) ? 'length' : ITERATE_KEY
      addRunners(effects, computedRunners, depsMap.get(iterationKey))
    }
  }
  // 遍历执行依赖的effect
  const run = (effect: ReactiveEffect) => {
    scheduleRun(effect, target, type, key, extraInfo)
  }
  computedRunners.forEach(run)
  effects.forEach(run)
}

function scheduleRun(
  effect: ReactiveEffect,
  target: any,
  type: OperationTypes,
  key: string | symbol | undefined,
  extraInfo: any
) {
  ...
  if (effect.scheduler !== void 0) {
    effect.scheduler(effect)
  } else {
    effect()
  }
}
```

由源码我们可以分析出：1、判断key是否为当前被代理对象target自身属性;  2、判断旧值与新值是否相等。只有这两个条件其中一个满足，才有可能执行 <code>trigger</code>。

怎么理解呢，我们举个🌰，可以实现一个小的 <code>reactive</code> 方法来做数据代理，代码如下：

```js
function hasOwn(val, key) {
  const hasOwnProperty = Object.prototype.hasOwnProperty
  return hasOwnProperty.call(val, key)
}
function reactive(data) {
  let observed = new Proxy(data, {
    get(target, key, receiver) {
      const res = Reflect.get(target, key, receiver)
      return res
    },
    set(target, key, value, receiver) {
      console.log(target, key, value)
      const hadKey = hasOwn(target, key)
      const oldValue = target[key]

      const result = Reflect.set(target, key, value, receiver)
      if (!hadKey) {
        console.log('trigger add operation...')
      } else if(value !== oldValue) {
        console.log('trigger set operation...')
      }

      return result
    }
  })
  return observed
}

let data = ['a', 'b']
let state = reactive(data)
state.push('c')

// ["a", "b"] "2" "c"
// trigger add operation...
// ["a", "b", "c"] "length" 3

```

<code>state.push('c')</code> 会触发两次 <code>set</code> ，一次是push的值 <code>c</code> ，一次是 <code>length</code> 属性设置。  
1、设置值 <code>c</code> 时，新增了索引 <code>key</code> 为 2，<code>target</code> 是原始的代理对象 <code>['a', 'c']</code> ，这是一个 <code>add</code> 操作, 故 <code>hasOwn(target, key)</code> 返回的是false，此时执行 <code>trigger add operation...</code> 。注意在 <code>trigger</code> 方法中，<code>length</code> 没有对应的 <code>effect</code>，所以就没有执行相关的 <code>effect</code>。  
2、当传入 <code>key</code> 为 <code>length</code> 时，<code>length</code> 是自身属性，故 <code>hasOwn(target, key)</code> 返回 <code>true</code>, 此时 <code>value</code> 是 3, 而 <code>oldValue</code> 即为 <code>target['length']</code> 也是 3，故 <code>value !== oldValue</code> 不成立，不执行 <code>trigger</code> 方法  
故只有当 <code>hasOwn(target, key)</code> 返回true或者 <code>value !== oldValue</code> 的时候才执行 <code>trigger</code>。

## 总结

在分析源码之前我们先列举了用Proxy做代理实现数据响应需要解决的几个问题，并带着这些问题一步一步揭开Vue在数据响应系统处理这些问题的面纱，也让我们进一步了解了Vue源码编写有许多巧妙的地方，比如利用 <code>Reflect.get</code> 返回值为 <code>target</code> 当前触发的第一层属性 <code>key</code> 值对应的 <code>value</code> 值，从而再来判断是否为Object来进行深度观测，并且观测的值存放在一个<code>WeakMap</code>下，这样相比较递归Proxy，Vue的这种实现方式大大提高了数据响应的性能。  
这两天就大致的阅读了这部分源码，有些细节地方还没去深入理解，存在不足或者错误的地方请指出，大家一起学习交流💪