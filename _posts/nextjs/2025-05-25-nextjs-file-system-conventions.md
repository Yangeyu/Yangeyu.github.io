---
layout: post
title: Next.js入土计划04-文件系统与你的约定
subtitle: 深入理解 Next.js App Router 的并行路由机制与关键文件约定：实现动态布局与健壮路由体验
author: Vascent
header-img: 
tags:
  - Nextjs
---

## **Parallel Routes(并行路由)**

并行路由使您可以在同一布局中同时或有条件地渲染一个或多个页面。它们对于应用程序的高度动态部分，例如仪表板和社交网站上的供稿很有用。

### Slot（插槽）定义与渲染
- **概念**：Parallel Routes 通过命名插槽（slot）实现「同一路由层级，多组件并行渲染」。
	
- **文件约定**：在 `app/[article]` 目录下，以 `@<slotName>/` 命名文件夹创建插槽。例如：
```cpp
app
├── @analytics
│   └── page.tsx
├── @sidebar
│   └── page.tsx
├── [post]
│   └── page.tsx
├── layout.tsx
├── not-found.tsx
└── page.tsx
```

- **渲染方式**：父级 `layout.tsx` 接收三个 prop：`children`（隐式主插槽）、`sidebar`、`analytics`。你可以在布局中任意位置并行渲染它们：
```tsx
// app/layout.tsx
export default function RootLayout({
  children,
  sidebar,
  analytics,
}: Props) {
  return (
    <html lang="en">
      <body
        className={`${geistSans.variable} ${geistMono.variable} antialiased`}
      >
        {children}
        {sidebar}
        {analytics}
      </body>
    </html>
  );
}
```
**URL 不变**：`@sidebar`、`@analytics` 并不影响 URL，访问 `/aticle` 时，这两个插槽都会渲染对应的组件。

### 活跃状态与导航行为 [Next.j](https://nextjs.org/docs/app/building-your-application/routing/parallel-routes)
https://nextjs.org/docs/app/building-your-application/routing/parallel-routes
- **Soft Navigation（软导航）**
    - 客户端路由跳转时，Next.js 会 **保留各插槽的 React 树**，只替换对应插槽的新页面，不会卸载其他插槽。
    ![Kapture 2025-05-25 at 14.56.40.gif](https://raw.githubusercontent.com/Yangeyu/img-hoist/main/Kapture%202025-05-25%20at%2014.56.40.gif)

- **Hard Navigation（硬导航／刷新）**
    - 全页刷新时，Next.js 无法从 URL 判断哪些插槽应激活，这时它会寻找每个插槽下的 `default.js`：
        - 若存在，则渲染 `default.js` 作为回退。
        - 若不存在，则抛出 404。
	 ![Kapture 2025-05-25 at 15.06.29.gif](https://raw.githubusercontent.com/Yangeyu/img-hoist/main/Kapture%202025-05-25%20at%2015.06.29.gif)

### 3. `default.js` 回退组件 [Next.js](https://nextjs.org/docs/app/building-your-application/routing/parallel-routes)

- **用途**：当硬导航到某条子路由，但某些插槽没有对应页面时，用 `default.js` 提供回退 UI，避免 404 或空白## 3. `default.js` 回退组件 [Next.js](https://nextjs.org/docs/app/building-your-application/routing/parallel-routes)

- **用途**：当硬导航到某条子路由，但某些插槽没有对应页面时，用 `default.js` 提供回退 UI，避免 404 或空白
```cpp
app
├── @analytics
│   ├── default.tsx
│   └── page.tsx
├── @sidebar
│   ├── default.tsx
│   └── page.tsx
├── [post]
│   └── page.tsx
├── layout.tsx
└── page.tsx
```

![Kapture 2025-05-25 at 15.05.09.gif](https://raw.githubusercontent.com/Yangeyu/img-hoist/main/Kapture%202025-05-25%20at%2015.05.09.gif)


### 4. 读取当前激活段 (`useSelectedLayoutSegment(s)`)

```tsx
const segment = useSelectedLayoutSegment('analytics')
```
**用途**：在客户端组件中读取某个插槽当前渲染的子路由段（segment）名称，便于高亮导航、条件渲染等。

## **error.js**
https://nextjs.org/docs/app/api-reference/file-conventions/error
### 1. 作用与概念 [nextjs.org](https://nextjs.org/docs/app/api-reference/file-conventions/error)

- **Error Boundary**：任何放在同一路由段（route segment）下的 `error.js` 文件，会被 Next.js 自动包裹在一个 React Error Boundary 中。
    
- **运行时错误处理**：当该路由段或其子组件在渲染时抛出异常，Error Boundary 会捕获它，并渲染你在 `error.js` 中定义的回退界面，而不是让整个应用崩溃。

### 2. 文件位置

```tsx
app/
└─ dashboard/
   ├─ page.tsx
   └─ error.tsx   ← 路由段级别的错误边界
```
- `error.js`/`error.tsx` 必须与同级的 `page.js` 或 `layout.js` 在同一目录下。
- 也可在更顶层目录（如 `app/error.js`）放置全局错误边界。

### 3. 必须是 Client Component

```tsx
'use client'  // 错误边界组件必须是客户端组件
```
React 的 Error Boundary 只能在客户端生效，因此 `error.js` 文件首行要加上 `'use client'`。

### 4. 导出组件签名

```tsx
export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) { /* ... */ }
```

- **`error`**
    - 捕获到的 `Error` 实例。
    - 在开发环境下包括完整 `message`；生产环境下仅保留通用信息，并提供 `digest`（hash）用于日志映射。
        
- **`reset`**
    - 一个函数，调用后会 **重新渲染** 当前路由段。
    - 适用于临时错误（如网络波动）时，用户点击「重试」后恢复页面
    - `reset` 本身是由 Next.js 在渲染 `error.js` 时注入的一个回调，用来清空当前路由段的错误状态并重新渲染。**无法替换它的底层实现**（它始终会触发 Next.js 重新执行该路由段的渲染逻辑）。

#### **测试验证**

1. **在子组件中故意抛错**  
在某个子页面或子组件里插入一个抛错语句，观察是否被 `error.js` 捕获并渲染。
```tsx
// app/dashboard/page.tsx
export default function DashboardPage() {
  throw new Error('测试错误！');
  return <h1>Dashboard</h1>;
}
```

```tsx
// app/dashboard/error.tsx
'use client'
import { Button } from "@/components/ui/button";

export default function DashboardError({ error, reset }: { error: Error, reset: () => void }) {
  return (
    <div className="bg-red-500">
      Dashboard Error {error.message}
      <Button onClick={reset}>Reset</Button>
    </div>
  );
}
```

2. **访问对应路由**  
启动开发服务器（`npm run dev`），打开浏览器访问 `/dashboard`：

- **预期**： `app/dashboard/error.tsx` 组件界面出现。
- 点击“重试”按钮，`reset()` 会重新渲染该路由段，若错误依旧，则再次触发错误边界。

## `loading.js` 
https://nextjs.org/docs/app/api-reference/file-conventions/loading
### 一、`loading.js` 的作用与放置位置

- **文件功能**：在路由切换或异步数据加载过程中，立即渲染一个“占位”或“骨架”界面，待真正内容准备好后自动替换。
- 
- **基于 Suspense**：Next.js 会在渲染对应 `page.js` 或其子层级前，将同级的 `loading.js` 包裹在 React 的 `<Suspense>` 中，快速显示加载状态，再替换成真实内容。
- 
- **放置位置**：在任意路由段目录下创建 `loading.js`（或 `.tsx`），与 `page.js`、`layout.js` 同级：
```tsx
app/
└─ loading-demo/
   ├─ page.tsx
   └─ loading.tsx   ← 只需导出一个组件，无参数
```

- **组件类型**：默认是 Server Component，也可以加上 `'use client'` 将其作为 Client Component，来使用客户端钩子或更丰富的动画。
    
- **签名**：不接收任何 props，单纯返回一个 React 节点：
```tsx
export default function Loading() {
  return <p>Loading…</p>
}
```

### 验证测试
1. 加载状态
```tsx
// app/loading-demo/loading.tsx
// 可加 'use client' 来启用客户端动画或钩子
export default function Loading() {
  return <div className="bg-yellow-500">Loading...</div>;
}
```

2. 延迟渲染页面：`app/loading-demo/page.tsx`
```tsx
export default async function LoadingDemo() {
  await new Promise((resolve) => setTimeout(resolve, 1000));
  return <div className="bg-blue-500">Loading Demo</div>;
}
```

![Kapture 2025-05-25 at 16.36.32.gif](https://raw.githubusercontent.com/Yangeyu/img-hoist/main/Kapture%202025-05-25%20at%2016.36.32.gif)


## **not-found.js**

https://nextjs.org/docs/app/api-reference/file-conventions/not-found

### 1. 作用与行为
- **用途**：当在某个路由段内部调用 `notFound()`（来自 `next/navigation`）或用户访问了一个未匹配的 URL 时，Next.js 会渲染对应目录下的 `not-found.js` 组件，用于提供自定义的 404 页面。
    
- **HTTP 状态**：
    - **流式响应（Streaming）**：返回 `200`，因为它是流式渲染的一部分。
    - **非流式响应**：返回 `404`，符合标准 404 行为。
### 2. 放置位置
```tsx
app/
├─ dashboard/
│  ├─ page.js
│  ├─ not-found.js    ← 针对 `/dashboard/...` 内部的 404
│  └─ ...
└─ not-found.js       ← 根级全局 404，捕获所有未匹配路径
```

- 在特定路由目录下的 `not-found.js` 只作用于该目录及其子路由。
- 根目录下的 `app/not-found.js` 则作为全站的兜底 404。
### 3. 组件签名与特性

- **无 props**：`not-found.js` 组件不接受任何 props。
- **Server Component**：默认是服务器组件，允许使用 `async` 并在其中进行数据获取。
- **可发起数据请求**：可标记为 `async` 并调用 `headers()`、`cookies()` 等服务端 API 获取上下文信息

```tsx
// app/not-found.tsx
export default function NotFound() {
  return (
    <div className="p-8 text-center">
      <h2 className="text-2xl font-bold">页面未找到</h2>
      <p className="mt-4">抱歉，我们找不到您访问的页面。</p>
      <a href="/" className="mt-6 inline-block text-blue-600">
        返回首页
      </a>
    </div>
  )
}
```
这里测试用 `Link` 组件点击跳转会出现问题。有 **Parallel Routes** 这玩意时存在下面的报错。 
![image.png](https://raw.githubusercontent.com/Yangeyu/img-hoist/main/20250525171951135.png)
#### unauthorized.js与之类似


Demo: https://github.com/Yangeyu/nextjs-tutorial/tree/main/lesson-05-file-sys-conventions

## 节外生枝
 **LLaMA-Factory** 是一个功能强大、结构清晰的 **大语言模型（LLM）微调框架**，主要目标是简化 LLaMA、ChatGLM、Baichuan、Qwen 等主流开源模型的 **高效微调（如 LoRA / QLoRA）与部署**流程。

| 特性     | 简介                                                     |
| ------ | ------------------------------------------------------ |
| 多模型支持  | 支持 Qwen, LLaMA, Baichuan, ChatGLM, InternLM, Mistral 等 |
| 多种训练范式 | 支持全量微调、LoRA、QLoRA、Prefix Tuning、AdaLoRA 等              |
| 高效微调   | 基于 Hugging Face + PEFT + bitsandbytes，内存占用少，适合消费级显卡    |
| 数据格式兼容 | 支持 Alpaca, ShareGPT, ChatML, Baize, Belle 等常见数据格式      |
| 推理部署   | 支持 CLI 推理、Web UI (基于 Gradio)、API Server、OpenAI 接口模拟等   |
| 快速上手   | 提供开箱即用的训练脚本与配置文件                                       |
| 可复现性   | 每个训练过程都使用统一 seed，可复现实验结果                               |

![Kapture 2025-05-25 at 17.52.14.gif](https://raw.githubusercontent.com/Yangeyu/img-hoist/main/Kapture%202025-05-25%20at%2017.52.14.gif)

http://github.com/hiyouga/LLaMA-Factory/tree/main