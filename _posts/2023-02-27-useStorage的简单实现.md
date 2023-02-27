---
layout: post
title: "useStorage的简单实现"
subtitle: "根据vueuse简单实现一个useStorage"
author: "Vascent"
header-img: "//w.wallhaven.cc/full/5g/wallhaven-5gylo8.jpg"
tags:
  - hooks
  - vue
---

# useStorage的简单实现

### 目标

- 第一次运行设置初始值，而后读取本地数据返回
- 数据发生更新时，同步更新本地数据
- 当其他标签页更改本地数据时，同时更新响应式数据

### 初始化本地数据

- 第一次运行设置初始值
- 而后读取本地数据返回

```jsx
const val = ref<any>(null)
const lval = localStorage.getItem(key)
if (!lval) {
  val.value = initValue
  localStorage.setItem(key, JSON.stringify(initValue))
} else {
  val.value = JSON.parse(lval)
}
```

### 监听响应式数据

> *💡 数据发生更新时，同步更新本地数据*
> 

```jsx
const { pause, resume } = pausableWatch(val, (newVal) => {
  const serialized = JSON.stringify(newVal)
  localStorage.setItem(key, serialized)
}, {
  deep: true
})
```

> *💡* `*pausableWatch` 返回控制一对控制暂停和继续的方法，供 `storage` 事件回调使用*
> 

```jsx
const pausableWatch = (source: any, cb: (...args: any[]) => void, options: any) => {
	const {eventFilter: filter, ...watchOptions} = options
	const { pause, resume, eventFilter } = pausableFilter(filter)
	const stop = watchWithFilter(source, cb, {...watchOptions, eventFilter})
	return {pause, resume, eventFilter, stop}
}

// 将filter包装一个暂停和继续的判断
const pausableFilter = (filter = bypassFilter) => {
  const isActive = ref(true)
  const pause = () => { isActive.value = false }
  const resume = () => { isActive.value = true }
  const eventFilter = (fn: any) => { if (isActive.value) filter(fn) }
  return {pause, resume, isActive, eventFilter}
}

// 用filter包装cb的执行: 
const watchWithFilter = (source: any, cb: any, options: any) => {
  const {
    eventFilter = bypassFilter,
    ...watchOptions
  } = options
  return watch(
    source,
    createFilterWrapper(eventFilter, cb),
    watchOptions
  )
}

const bypassFilter: (invoke: any)=> any = (invoke) => invoke()

function createFilterWrapper (eventFilter: any, cb: any) {
  function wrapper(this: any, ...args: any[]) {
    console.log('this: ', this);
    eventFilter(() => cb.apply(this, args))
  }
  return wrapper
}
```

### 绑定`storage`事件

> *💡 当其他标签页更改本地数据时，更新响应式数据*
> 

```jsx
window.addEventListener('storage', (ev) => {
    pause()
  const newValue = ev.newValue
  if (!newValue) {
    localStorage.setItem(key, JSON.stringify(initValue))
    val.value = initValue
  } else {
    val.value = JSON.parse(newValue)
  }
  resume()
}, false)
```
