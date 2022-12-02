---
title: vue3-composition-api
catalog: true
comments: true
indexing: true
header-img: ../../../../img/default.jpg
top: false
tocnum: true
date: 2022-12-02 10:09:41
subtitle:
tags:
- vue3
categories:
- composition
---

# Vue3 响应式原理

## ref & reactive

`ref`使用`getter / setter` 劫持数据。

```js
function ref(value) {
    const refObject= {
        get value() {
            track(refObject, 'value');
            return value;
        },
        set value(newValue) {
            value = newValue;
            trigger(refObject, 'value');
        }
    }
    return refObject;
}
```

`reactive`使用 `Proxy` 劫持数据。

```js
function reactive (obj) {
    return new Proxy(obj, {
        get(target, key) {
            track(target, key);
            return target[key];
        },
        set(target, key, value) {
            target[key] = value;
            trigger(target, key);
        }
    })
}
```

## track 跟踪

```js
let activeEffect;

function track(target, key) {
    if ( activeEffect ) {
        const effects = getSubscriberForProperty(target, key);
        effects.add(activeEffect);
    }
}
```

## trigger 触发

```js
function trigger(target, key) {
    const effects = getSubscriberForProperty(target, key);
    effects.forEach(effect => effect());
}
```

## watchEffect  响应式副作用

```js
import { ref, watchEffect } from 'vue'

const A0 = ref(0)
const A1 = ref(1)
const A2 = ref()

watchEffect(() => {
  // 追踪 A0 和 A1
  A2.value = A0.value + A1.value
})

// 将触发副作用
A0.value = 2
```

`ref`或者`reactive`创建响应式对象,通过`watchEffect`获取响应式副作用,从而达到响应式的目的。

# 响应式的核心

## ref

- 接收任何类型的值
- 返回一个`get/set`包含响应式的`.value`的对象
- 如果`value`是集合类型会自动使用`reactive`进行响应式的转化

### ref的解包

- 作为顶层属性被访问时,会自动解包,不需要`.value`,如果不做为上下文的顶级属性时,不会自动解包
- 在响应式对象中会自动解包，表现得跟一般属性一致

```js
const count = ref(0)
const state = reactive({
  count
})

console.log(state.count) // 0

state.count = 1
console.log(count.value) // 1
```

- `ref`在数组和集合的响应式类型中不会解包

```js
const books = reactive([ref('Vue 3 Guide')])
// 这里需要 .value
console.log(books[0].value)

const map = reactive(new Map([['count', ref(0)]]))
// 这里需要 .value
console.log(map.get('count').value)
```

## reactive

- 接收 `array`, `object`, `map`, `set` 这样的对象集合类型
- vue通过属性的响应式追踪,如果更改了对象集合类型的引用，会导致响应式的丢失

```ts
const state = reactive({ count: 0 })

// n 是一个局部变量，同 state.count
// 失去响应性连接
let n = state.count
// 不影响原始的 state
n++

// count 也和 state.count 失去了响应性连接
let { count } = state
// 不会影响原始的 state
count++

// 该函数接收一个普通数字，并且
// 将无法跟踪 state.count 的变化
callSomeFunction(state.count)

```

## computed 计算属性

- `computed`期望接收一个`getter`函数,返回一个 **计算属性的ref**
- `computed`自动追踪函数内的响应式依赖
- `computed`计算属性基于响应式依赖的缓存,当依赖没有被改变,不会去主动执行`getter`函数
- `computed`默认只有`get`,我们也可以设置`set`变成一个可写的计算属性
- `computed`的`get`中不要包含其他的副作用, 请求以及操作DOM

## watchEffect 侦听器

立即运行一个函数,同时响应式的跟踪其依赖,并当依赖更新时重新运行函数

### 类型

```ts
function watchEffect(
  effect: (onCleanup: OnCleanup) => void,
  options?: WatchEffectOptions
): StopHandle

type OnCleanup = (cleanupFn: () => void) => void

interface WatchEffectOptions {
  flush?: 'pre' | 'post' | 'sync' // 默认：'pre'
  onTrack?: (event: DebuggerEvent) => void
  onTrigger?: (event: DebuggerEvent) => void
}

type StopHandle = () => void
```

- `effect`: 要执行的副作用函数,这个函数也包含一个参数,这个参数再下一次调用前被执行,用于执行清除无效的副作用
- `options`: 可选参数,用来调整副作用的刷新时机以及调试依赖
  - `flush`: 侦听器执行时机
    - pre: 组件渲染前执行
    - post: 组件延迟到渲染后执行
    - sync: 响应依赖发生改变后立马执行
  - onTrack:
  - onTrigger:
- `StopHandle`: 停止副作用函数

## watch
