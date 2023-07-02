---
layout: post
title: "Vue.js响应式原理 - 数组拦截"
subtitle: '了解 Vue.js 响应式原理 - Array.prototype.push'
author: "Vascent"
header-img: "//w.wallhaven.cc/full/k7/wallhaven-k7q9m7.png"
<!-- header-style: text -->
tags:
  - Vue
---

### Vue响应式原理-数组push的拦截

```typescript
const arr = reactive([0])

watchEffect(() => {
  console.log(arr.length)
})

arr.push(1)

```

- 执行watchEffect，收集了关于 `arr.length` 的依赖
- 执行 `arr.push(1)` 动作的时候，触发了依赖的执行

### **了解push方法的内部执行**

![imgage-1](https://s2.loli.net/2023/02/22/4mw7VSUagMhKsnY.png)

- 读取`length`属性 -> 调用`track`方法 -> 建立`arr.length`跟`effectFn`的响应式连接
- 设置`length`值加1 -> 调用`trigger`方法 -> 取出`length`的依赖`effectFn`并执行

### **搭建一个简单的响应**

- 代码清单
  
    ```jsx
    // 存储收集的依赖
    const bucket = new WeakMap<object, Map<string, Set<any>>>()
    // 当前执行副作用函数
    let curActiveEffect: any = null
    const arr = mReactive([0])
    const o1 = mReactive({ a: 'aaa' })
    
    export function useTestV1() {
      mWatch(() => {
        console.log('xxxx', o1.a);
      })
      o1.a = 'bbbb'
    }
    
    // 数据代理
    function mReactive(target: any): any {
      return new Proxy(target, {
        get(target, key: string, receiver) {
          track(target, key)
          const res = target[key]
          return res
        },
        set(target, key: string, value) {
          target[key] = value
          trigger(target, key)
          return true
        },
      })
    }
    
    // 函数监听
    const mWatch = (effectFn: Function) => {
      curActiveEffect = effectFn
      effectFn()
      curActiveEffect = null
    }
    
    // 依赖收集
    const track = (target: any, key: string) => {
      if (!curActiveEffect) return
      const dep = safeGetFromBucket(target)
      const fns = safeGet(dep, key)
      fns.add(curActiveEffect)
    }
    
    // 触发依赖执行
    const trigger = (target: any, key: string) => {
      const dep = safeGetFromBucket(target)
      const effects = safeGet(dep, key)
      effects.forEach(fn => fn())
    }
    
    const safeGetFromBucket = (key: any) => {
      let dep = bucket.get(key)
      if (!dep) {
        dep = new Map()
        bucket.set(key, dep)
      }
      return dep
    }
    const safeGet = (dep: Map<string, Set<any>>, key: string) => {
      let fns = dep.get(key)
      if (!fns) {
        fns = new Set()
        dep.set(key, fns)
      }
      return fns
    }
    ```
    
- 拦截push方法
  
    ```jsx
    export function useTestV1() {
    	const arr = mReactive([0])
      mWatch(() => {
        console.log('watch...', arr.length)
        arr.push(2)
      })
      arr.push(1)
    }
    
    function mReactive(target: any): any {
      return new Proxy(target, {
        get(target, key: string, receiver) {
          // 拦截数组的push方法
    			if (Array.isArray(target) && key === 'push') {
    	      return (...args: any[]) => { Array.prototype.push.apply(receiver, args) }
          }
          track(target, key)
          const res = target[key]
          return res
        },
        set(target, key: string, value) {
          target[key] = value
          trigger(target, key)
          return true
        },
      })
    }
    
    ..... 代码省略 .....
    ```
    
    - `Array.prototype.push.apply(receiver, args)` 将 `this` 与代理对象进行绑定，(receiver指代代理对象)
    - `push` 方法在访问 `length` 属性的时候，是通过代理对象进行访问的
- 处理无限递归循环
  
    ```jsx
    ..... 代码省略 .....
    const mWatch = (effectFn: Function) => {
      const effect = () => {
        curActiveEffect = effect
        effectFn()
        curActiveEffect = null
      }
      effect()
    }
    
    const trigger = (target: any, key: string) => {
      const dep = safeGetFromBucket(target)
      const effectFns = safeGet(dep, key)
      // 过滤掉当前正在执行的副作用函数
      const runActiveFns = [...effectFns].filter(item => item !== curActiveEffect)
      runActiveFns.forEach(fn => fn())
    }
    ..... 代码省略 .....
    ```
    
    > 💡 在触发副作用依赖提取的时候，过滤掉当前正在执行的副作用方法
    
- 屏蔽push方法的依赖收集
  
    ```jsx
    // 设置是否应该进行副作用依赖收集
    let shouldTrack = true
    function mReactive(target: any): any {
      return new Proxy(target, {
        get(target, key: string, receiver) {
    			if (Array.isArray(target) && key === 'push') {
    
    	      return (...args: any[]) => {
              // 执行push方法屏蔽依赖收集 
    					shouldTrack = false
    					Array.prototype.push.apply(receiver, args) 
    					shouldTrack = true
    				}
          }
          shouldTrack && track(target, key)
          const res = target[key]
          return res
        },
        set(target, key: string, value) {
          target[key] = value
          trigger(target, key)
          return true
        },
      })
    }
    ..... 代码省略 .....
    ```
    
- 最终代码清单
  
    ```jsx
    // 存储收集的依赖
    const bucket = new WeakMap<object, Map<string, Set<any>>>()
    // 当前执行副作用函数
    let curActiveEffect: any = null
    // 用来判断是否进行依赖收集
    let shouldTrack = true
    // 存放拦截数组方法集合
    const arrayInstrumentions = createArrayInstrumentions()
    const arr = mReactive([0])
    
    export function useTestV3() {
      mWatch(() => {
        console.log('watch...', arr.length)
    
        // console.log('.........');
        // arr.push(1)
      })
    
      arr.push(2)
    }
    
    // 数据代理
    function mReactive(target: any): any {
      return new Proxy(target, {
        get(target, key: string, receiver) {
          if (Array.isArray(target) && Object.hasOwn(arrayInstrumentions, key)) {
            return Reflect.get(arrayInstrumentions, key, receiver)
          }
    
          if (shouldTrack) { track(target, key) }
          const res = target[key]
          return res
        },
        set(target, key: string, value) {
          target[key] = value
          trigger(target, key)
          return true
        },
      })
    }
    
    // 创建一个方法拦截的映射表
    function createArrayInstrumentions() {
      const instrumentions: Record<string, any> = {} as {}
    
      ['push'].forEach((key: string) => {
        instrumentions[key] = function (...args: any[]) {
          const originMethod = Array.prototype[key as any]
          // 执行该动作不进行依赖收集
          shouldTrack = false
          const res = originMethod.apply(this, args)
          shouldTrack = true
          return res
        }
      })
      return instrumentions
    }
    
    // 函数监听
    const mWatch = (effectFn: Function) => {
      const effect = () => {
        curActiveEffect = effect
        effectFn()
        curActiveEffect = null
      }
      effect()
    }
    
    // 依赖收集
    const track = (target: any, key: string) => {
      const dep = safeGetFromBucket(target)
      const fns = safeGet(dep, key)
      fns.add(curActiveEffect)
    }
    
    // 触发依赖执行
    const trigger = (target: any, key: string) => {
      const dep = safeGetFromBucket(target)
      const effectFns = safeGet(dep, key)
      // 过滤掉当前正在执行的副作用函数
      const runActiveFns = [...effectFns].filter(item => item !== curActiveEffect)
      runActiveFns.forEach(fn => fn())
    }
    
    const safeGetFromBucket = (key: any) => {
      let dep = bucket.get(key)
      if (!dep) {
        dep = new Map()
        bucket.set(key, dep)
      }
      return dep
    }
    const safeGet = (dep: Map<string, Set<any>>, key: string) => {
      let fns = dep.get(key)
      if (!fns) {
        fns = new Set()
        dep.set(key, fns)
      }
      return fns
    }
    ```
