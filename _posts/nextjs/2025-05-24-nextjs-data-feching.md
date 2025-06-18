---
layout: post
title: Next.js入土计划03-数据获取&SWR
subtitle: Data Fetching & SWR
author: Vascent
header-img: 
tags:
  - Nextjs
---

## 服务端数据获取和客户端数据获取的差异对比

先在本地3001端口启用后端接口模拟数据
```json
const items = [
	{ id: 1, name: 'Item 1', description: 'Description for item 1' },
	{ id: 2, name: 'Item 2', description: 'Description for item 2' },
	{ id: 3, name: 'Item 3', description: 'Description for item 3' },
	{ id: 4, name: 'Item 4', description: 'Description for item 4' },
	{ id: 5, name: 'Item 5', description: 'Description for item 5' },
];
```

### 服务端数据获取

```tsx
// app/server/page.tsx
import { Button } from "@/components/ui/button";

const fetchItems = async () => {
  const res = await fetch('http://localhost:3001/api/items')
  const {data} = await res.json()
  console.log(data)
  return data
}
export default async function ServerPage() {

  const items = await fetchItems()
  return (
    <div className="flex flex-col gap-4 items-center">
      <h1>Server Page</h1>
      <main className="p-4 flex flex-col gap-4">
        {items.map((item: any) => (
          <div className="flex gap-4 items-center" key={item.id}>
            <Button>{item.name}</Button>
            <p>{item.description}</p>
          </div>
        ))}
      </main>
    </div>
  );
}
```

### 客户端数据获取

```tsx
//app/client/page.tsx
'use client'
import { Button } from "@/components/ui/button";
import { useEffect, useState } from "react";

export default function ClientPage() {
  const [items, setItems] = useState([])
  useEffect(() => {
    fetch('http://localhost:3001/api/items')
      .then(res => res.json())
      .then(data => setItems(data.data))
  }, [])

  return (...);
}
```


| 对比维度         | 服务端数据获取（Server Component）                                      | 客户端数据获取（Client Component）                       |
| ------------ | -------------------------------------------------------------- | ----------------------------------------------- |
| **组件类型**     | 默认（无需额外标记），是 React Server Component                            | 需在文件顶使用 `'use client'`，是 React Client Component |
| **执行环境**     | Node.js 或 Edge Runtime                                         | 浏览器端                                            |
| **数据获取时机**   | 渲染前：在组件渲染阶段同步执行 `await fetch()`                                | 渲染后：组件挂载后通过 `useEffect` / `useSWR` 等异步请求        |
| **缓存 & 重验证** | 通过 `fetch(url, { cache, next:{ revalidate } })` 支持 SSR、SSG、ISR | 依赖客户端缓存策略（SWR、React Query）或手动实现本地存储             |
| **首屏渲染体验**   | 页面初次加载即带数据，避免闪烁，SEO 友好                                         | 首屏可能显示 loading，再异步填充数据，SEO 无数据（直到 hydration 后）  |
| **SEO 支持**   | ✅ HTML 中即包含完整数据                                                | ❌ 数据由客户端 JS 渲染，不利于搜索引擎抓取                        |
| **敏感信息暴露**   | 可安全使用环境变量、私钥、服务端 SDK                                           | 不可使用任何后端凭证，只能调用公开 API                           |
| **错误处理**     | 可借助 `error.js`、`not-found.js` 在服务器渲染阶段统一捕获并渲染错误页               | 需在组件内部用状态管理（如 `error`、`isLoading`）并自行设计错误 UI    |
| **包体体积**     | 不会把数据获取逻辑打包到客户端，减小前端大小                                         | Fetch 库、状态管理逻辑等都要打包到客户端，略增前端体积                  |
| **适用场景**     | 首屏关键数据、SEO 需求、静态站点、增量更新（ISR）                                   | 用户交互触发、实时更新、下拉刷新、分页加载、长列表虚拟滚动等                  |

## **SWR**
SWR（stale-while-revalidate）是由 Vercel 团队推出的 React Hooks 数据获取库，致力于**简化客户端数据请求与缓存**、自动更新的流程。它遵循 HTTP RFC 中的 “stale-while-revalidate” 缓存策略：在展示旧数据（stale）的同时，后台发起请求并获取最新数据（revalidate），然后自动更新界面。
https://swr.vercel.app/zh-CN/docs/getting-started
	
### 核心特性
- **自动缓存**
    - 相同 key 的请求会被缓存，避免重复网络请求。
    - 默认使用内存缓存，也可以结合 `localStorage`、`IndexedDB` 等做持久化。
        
- **自动重验证（Revalidation）**
    - **窗口聚焦重新请求**：当浏览器窗口重新聚焦时，自动发起重验证，确保数据新鲜。
    - **网络恢复重新请求**：断网后恢复网络时，自动重验证。
    - **定时重新验证**：通过 `refreshInterval` 配置，可以定期自动拉取最新数据。
        
- **请求去抖与并发控制**
    - 同一时刻相同 key 只会发起一次请求，后续请求复用中间态结果并共享更新。
        
- **本地乐观更新（Optimistic UI）**
    - 在真正请求完成前，可先更新本地数据，通过 `mutate` API 手动控制，提升交互体验。
        
- **错误重试**
    - 默认失败自动重试（可自定义重试次数与间隔），并暴露错误给业务逻辑处理。
        
- **TypeScript 支持**
    - 完整的类型定义与泛型支持，方便在 TS 项目中安全使用

### 快速上手
1. **安装依赖包**
```shell
pnpm install swr
```

2. **全局配置**  
在项目根组件（如 `app/layout.tsx` 包裹 `<SWRConfig>`，统一设置：

编写`swr-provider.tsx`
```tsx
// app/components/swr-provider.tsx
'use client';

import { SWRConfig } from 'swr';
import { ReactNode } from 'react';

// Global fetcher function
const fetcher = (url: string) =>
  fetch(url).then(res => {
    if (!res.ok) { throw new Error('An error occurred while fetching the data.'); }
    return res.json();
  });

export function SWRProvider({ children }: { children: ReactNode }) {
  return (
    <SWRConfig
      value={{
        fetcher,
        revalidateOnFocus: true,
        revalidateOnReconnect: true,
        dedupingInterval: 5000,
        errorRetryCount: 3,
      }}
    >
      {children}
    </SWRConfig>
  );
} 
```

在`app/layout.tsx`中设置组件:
```tsx
...
<SWRProvider>
	{children}
</SWRProvider>
...
```

在客户端页面进行请求渲染：
```tsx
app/client-swr/page.tsx
'use client'
import { Button } from "@/components/ui/button";
import useSWR from "swr";

// Client SWR component that uses our custom hook
export default function ClientSWRContent() {
  // Using our custom hook for data fetching
  const { data, error, isLoading, mutate } = useSWR(`http://localhost:3001/api/items`);
  const items = data?.data || [];
  
  // Show loading state
  if (isLoading) return <div className="flex justify-center p-8">Loading...</div>;
  
  // Show error state
  if (error) return <div className="flex justify-center p-8 text-red-500">Failed to load data</div>;
  return ...
}
```

### 其他用法

- **依赖动态参数**
```tsx
const userId = 123;
const { data } = useSWR(userId ? `/api/user/${userId}` : null, fetcher);
```
当 `key` 为 `null` 时，SWR 不会发请求。

- **并行/串行请求**
```tsx
// 并行
const { data: post } = useSWR('/api/post/1', fetcher);
const { data: comments } = useSWR('/api/comments?postId=1', fetcher);

// 串行：后一个依赖前一个的数据
const { data: post } = useSWR('/api/post/1', fetcher);
const { data: comments } = useSWR(post ? `/api/comments?postId=${post.id}` : null, fetcher);
```

- **mutate乐观更新** 
```tsx
// app/components/item-form.tsx
  // 数据列表
  const { data, isLoading, error, mutate } = useSWR('http://localhost:3001/api/items');
  
  const [state, formAction] = useActionState(async (state: any, formData: any) => {
	// 提交表单
    const id = formData.get('id');
    const name = formData.get('name');
    const description = formData.get('description');
    const newItem = { id, name, description };
    // 提前更新数据列表
    mutate({data: data.data.map((item: any) => item.id == id ? newItem : item)}, false);
    // 发送更新数据请求
    await fetch('http://localhost:3001/api/items/update', {
      method: 'PUT',
      body: JSON.stringify(newItem),
    });
    // 重新校验数据列表
    mutate();
    return newItem;
  }, {
    id: 1,
    name: 'New Item 1',
    description: 'New Description 1',
  });
```
**解释**：
1. `mutate(newData, false)`：立即替换本地缓存为 `newData`，并且不触发网络请求。常用于乐观更新。
	- 第二个参数简单理解就是，决定当前 `mutate` 是否发送请求。
2. `mutate()`：默认重新发起请求，拿到最新数据后更新缓存并刷新界面。

### 何时使用 SWR

- 客户端渲染（CSR）或混合渲染场景，需要自动缓存、自动刷新、并发控制时。
- 交互频繁、依赖后端数据的页面，如评论列表、实时仪表盘、用户设置等。
- 希望在最少手写逻辑的前提下，实现高可用、可控的请求与缓存策略。
- 想用的时候

Demo: https://github.com/Yangeyu/nextjs-tutorial/tree/main/lesson-04-nextjs-data-fetching-swr

## 节外生枝
**`useActionState`** 是 React 19.1 中的新 Hook，专门用于简化基于表单动作（Form Action）的状态管理。它结合了 React Server Components 的表单动作能力，让你能够：
- 在表单提交前后持有并更新一段状态
- 在提交响应返回后，自动将返回值映射为最新状态
- 在服务器组件渲染阶段就能显示最新状态，提升无障碍和首屏体验 [React](https://react.dev/reference/react/useActionState)
	- https://react.dev/reference/react/useActionState

### API 概览
```tsx
const [state, formAction, isPending] = useActionState(
  actionFn,        // 由 useActionState 包装的表单动作函数
  initialState,    // 初始状态（任意可序列化值）
  permalink?       // （可选）表单动作对应的唯一页面 URL，用于 Progressive Enhancement
);
```

- **`actionFn`**  
    传入一个常规的表单动作函数。当表单提交或按钮触发时，React 会调用它，并将最新状态作为第一个参数、表单数据（`FormData`）作为后续参数 [React](https://react.dev/reference/react/useActionState)。
- **`initialState`**  
    首次渲染时的状态值。直到第 1 次提交前，它会作为 `state` 返回。
- **`permalink`**（可选）  
    在支持 React Server Components 的环境下，用于在 JS 未加载前，浏览器提交表单时跳转到指定的静态 URL，以保证 Progressive Enhancement 的兼容性 [React](https://react.dev/reference/react/useActionState)。

**返回值**
1. **`state`**：当前状态，首次为 `initialState`，之后为最新一次动作返回值。
2. **`formAction`**：新的动作函数，可作为 `<form action={formAction}>` 或 `<button formAction={formAction}>` 的值。
3. **`isPending`**：布尔值，表示有没有正在进行的表单动作（即 Transition 未完成）。