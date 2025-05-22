---
layout: post
title: Next.js路由机制和图片资源浅谈
subtitle: 从文件系统路由到智能图片加载：深度解析 App Router 架构与 <Image /> 组件的终极用法
author: Vascent
header-img: https://w.wallhaven.cc/full/9d/wallhaven-9d3181.jpg
tags:
  - Nextjs
---

## 路由机制
Next.js 的路由机制是基于文件系统自动生成的路由，支持两种主要模式：
### 1. 路由模式对比（App Router vs Pages Router）

| 特性    | **App Router（`/app`）**              | **Pages Router（`/pages`）**             |
| ----- | ----------------------------------- | -------------------------------------- |
| 目录    | `app/`                              | `pages/`                               |
| 文件    | `page.tsx`, `layout.tsx`            | 组件文件（如 `index.tsx`）                    |
| 数据加载  | `fetch()`, `React Server Component` | `getStaticProps`, `getServerSideProps` |
| 布局支持  | 原生多级嵌套 `layout.tsx`                 | 需要手动封装 `_app.tsx`                      |
| 状态保持  | 支持（导航不会重载布局）                        | 不支持（每次导航都重新渲染）                         |
| 动态路由  | `[slug]`, `[id]` 等                  | 同上                                     |
| 状态成熟度 | Next.js 13+ 新特性                     | 更加成熟，兼容旧系统                             |

### 2. App Router 路由机制

📁 路由通过 `app/` 目录结构定义：
```cpp
.
├── app                    # Next.js App Router 目录，包含页面和布局
│   ├── about             # About 页面相关目录
│   │   └── page.tsx      # About 页面的组件，定义页面内容
│   ├── blog              # 博客相关页面目录
│   │   └── [slug]        # 动态路由目录，支持基于 slug 的博客文章
│   │       └── page.tsx  # 博客文章页面组件，根据 slug 动态渲染
│   ├── favicon.ico       # 网站图标（建议移至 public/ 目录以符合 Next.js 约定）
│   ├── globals.css       # 全局 CSS 文件，定义整个应用的通用样式
│   ├── layout.tsx        # 根布局组件，定义所有页面的通用结构（如导航、页脚）
│   ├── not-found.tsx     # 404 页面组件，处理未找到的页面
│   └── page.tsx          # 首页组件，定义网站的主页内容
├── assets                # 静态资源目录（非 Next.js 标准，建议移至 public/）
│   └── next.svg          # Next.js 相关 SVG 文件（如 Logo）
├── components            # 可复用组件目录
│   └── ui                # UI 组件子目录
│       └── button.tsx    # 按钮组件，定义可复用的 UI 按钮
├── public                # 公共静态资源目录，文件可通过根路径访问
│   ├── globe.svg         # SVG 文件，可通过 /globe.svg 访问
└── tsconfig.json         # TypeScript 配置文件，定义 TypeScript 编译选项
```
### 3. 支持的路由类型

- 🧩 静态路由（静态文件结构）
```cpp
app/
└── about/
    └── page.tsx      → /contact
```

- 📦 动态路由（动态参数）
```cpp
app/
└── blog/
    └── [slug]/
        └── page.tsx  → /blog/abc，/blog/xyz
```
文件名中的 `[slug]` 就表示一个变量。

可以访问 `app/blog/[slug]/page.tsx` ：
```tsx
export default async function BlogPost({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  return div>Blog Post: {slug}</div>;
}
```

- 🌀 可选参数
```cpp
app/
└── blog/
    └── [[slug]]/
        └── page.tsx   → /blog or /blog/abc

```

### 4. 特殊文件的作用

| 文件名             | 作用               |
| --------------- | ---------------- |
| `layout.tsx`    | 定义当前路径及其子路径的公共布局 |
| `page.tsx`      | 页面内容，自动对应路由      |
| `error.tsx`     | 局部错误处理           |
| `loading.tsx`   | 路由加载状态 UI        |
| `not-found.tsx` | 404 页面           |
| `template.tsx`  | 每次进入都重新渲染的布局     |
### 5. 链接导航

```tsx
import { Button } from "@/components/ui/button";
import Link from "next/link";

export default function Home() {
  return (
    <div>
      <main className="flex flex-col gap-[32px] row-start-2 items-center sm:items-start">
        <Button variant="link" asChild>
          <Link href="/about">About</Link>
        </Button>
      </main>
    </div>
  );
}
```

## **图片资源**

### [`next/image`](https://nextjs.org/docs/app/api-reference/components/image)
Next.js 提供的内置 `<Image />` 组件。
文档地址：https://nextjs.org/docs/app/api-reference/components/image
它相较于原生 `<img>` 拥有以下优势：

| 优点          | 说明                  |
| ----------- | ------------------- |
| ✅ 自动压缩      | 构建时自动生成优化版本（如 WebP） |
| ✅ 自适应       | 根据设备尺寸提供不同图片大小      |
| ✅ 懒加载       | 默认开启懒加载，提升性能        |
| ✅ SEO 友好    | 支持 alt 属性           |
| ✅ 支持本地与远程图片 | 本地文件、外链都能使用         |

✅ 基本用法（用于 App Router）
```tsx
// app/page.tsx
import Image from 'next/image'
import nextLogo from '@/assets/next.svg' // 本地导入

export default function Page() {
  return (
    <Image src={nextLogo} alt="nextjs logo" width={100} height={100} />
  )
}
```

或使用 public 目录下的图片：
```tsx
<Image src="/globe.png" alt="globe logo" width={100} height={100} />
```

✅ 远程图片（需要配置）
在 `next.config.js` 中配置远程图片域名：

```js
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: "images.unsplash.com",
        port: "",
        pathname: "/**",
      },
    ],
  },
};

export default nextConfig;
```

在 `app/page.tsx` 中展示 `unsplash` 图片

```tsx
<Image
	src="https://images.unsplash.com/photo-1596367407372-96cb88503db6?q=80&w=3270&auto=format&fit=crop&ixlib=rb-4.1.0&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D"
	alt="random image"
	width={100}
	height={100}
	className="w-[100px] h-[100px] object-cover"
/>
```

> ⚠️ Tailwind preflight 有设置 `img { ... height: auto ...}`, 根据CSS优先级 height 属性值会被覆盖

✅ 自适应尺寸加载
```tsx
<Image
  src="/demo.jpg"
  alt="random image"
  width={100}
  height={100}
  className="w-[100px] h-[100px] object-cover"
  sizes="(max-width: 768px) 100vw, 50vw"
/>

```

🔧 这里的 `sizes` 属性告诉浏览器：

- 在屏幕宽度 ≤ 768px 时：这个图片显示占满 **100% 的视口宽度**  
- 否则：显示宽度为 **50% 的视口宽度**
📦 Next.js 会：

1. 在构建时自动生成多种尺寸版本（如 320px, 640px, 1024px…）
2. 浏览器根据设备情况从中选择最合适的版本下载 

✅ 手机上自动加载小图，桌面加载高清图，节省带宽，提升速度。

```html
<img
  sizes="(max-width: 768px) 100vw, 50vw"
  srcset="/hero-400w.jpg 400w, /hero-800w.jpg 800w, /hero-1600w.jpg 1600w"
  src="/hero-1600w.jpg"
/>
```
- `sizes`：告诉浏览器“当前视口条件下，图片会显示多宽”
- `srcset`：由 Next.js 自动生成，列出多个不同尺寸的图片 URL

Demo项目地址：https://github.com/Yangeyu/nextjs-tutorial/tree/main/lesson-02-nextjs-introduce-routing-image

## 节外生枝

`tree` 是一个用于在终端中**以树状结构**展示目录内容的命令行工具

| 指令                | 含义                    |
| ----------------- | --------------------- |
| `tree`            | 显示当前目录及其所有子目录、文件的树状结构 |
| `tree -L N`       | 限制显示 **N 层** 目录（常用）   |
| `tree -d`         | 仅显示目录（不显示文件）          |
| `tree -I PATTERN` | 忽略匹配模式的文件/目录（支持通配符）   |
| `tree -P PATTERN` | 仅显示匹配模式的文件/目录         |
| `tree -h`         | 显示文件大小并自动加单位（K, M, G） |
| `tree -s`         | 显示文件大小（以字节为单位）        |
| `tree -i`         | 以纯文本（无图形符号）方式显示目录结构   |

