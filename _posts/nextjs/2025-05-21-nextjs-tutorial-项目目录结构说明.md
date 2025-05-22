---
layout: post
title: NextJs Tutorial - 项目初始化与目录结构详解
subtitle: 项目的结果目录风格建议
author: Vascent
tags:
  - Nextjs
header-img: https://w.wallhaven.cc/full/ly/wallhaven-ly9qzq.jpg
---

## 项目初始化

```shell
npx create-next-app@latest
```

后面你会看到项目的配置提示


```shell
What is your project named? my-app
Would you like to use TypeScript? No / Yes
Would you like to use ESLint? No / Yes
Would you like to use Tailwind CSS? No / Yes
Would you like your code inside a `src/` directory? No / Yes
Would you like to use App Router? (recommended) No / Yes
Would you like to use Turbopack for `next dev`?  No / Yes
Would you like to customize the import alias (`@/*` by default)? No / Yes
What import alias would you like configured? @/*
```

如何结合`shadcn`使用可以使用 `shadcn` 的脚手架工具来初始化项目

```shell
pnpm dlx shadcn@latest init
```

## 🧭 推荐项目结构总览

```python
my-app/
├── app/                     # App Router 路由目录
│   ├── layout.tsx          # 全局布局组件
│   ├── page.tsx            # 首页页面
│   ├── about/              # 路由 segment - 对应 /about 页面
│   │   ├── page.tsx
│   │   └── layout.tsx
│   ├── dashboard/          # 嵌套路由示例
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   └── settings/
│   │       └── page.tsx
│   ├── api/                # API 路由（用于处理请求）
│   │   └── route.ts
│   └── not-found.tsx       # 自定义 404 页面
│
├── components/             # 全局可复用组件
│   ├── Header.tsx
│   ├── Footer.tsx
│   └── Button.tsx
│
├── lib/                    # 工具函数和服务模块，如数据库/网络请求逻辑
│   ├── db.ts
│   └── auth.ts
│
├── styles/                 # 全局样式、Tailwind 配置等
│   ├── globals.css
│   └── tailwind.config.js
│
├── public/                 # 静态资源文件（直接通过 URL 访问）
│   ├── favicon.ico
│   └── images/
│       └── logo.png
│
├── middleware.ts           # 中间件逻辑
├── next.config.js          # Next.js 配置
├── tsconfig.json           # TypeScript 配置（如果使用 TS）
└── package.json            # 项目信息与依赖

```

---

## 📁 各目录详解与最佳实践

### `app/`（核心目录，App Router）

- **每个子目录都是一个路由段（segment）**

- 包含：
    
    - `page.tsx`：定义页面
        
    - `layout.tsx`：定义该路由层的布局
        
    - `loading.tsx`：页面加载状态
        
    - `error.tsx`：错误处理页面
        
    - `not-found.tsx`：404 页面
        
    - `route.ts`：处理 API 路由（在 `api/` 子目录中）

✅ **最佳实践：**

- 按页面划分结构，每个页面有自己的子目录
    
- 把每层 layout 独立出来，灵活组合布局
    
- 把页面私有组件放在对应目录中，避免全局污染

---

### `components/`（可复用组件库）

- 全站通用的 UI 组件，如按钮、导航栏、卡片等

✅ **最佳实践：**

- 使用统一的命名规则（如 PascalCase）
    
- 可根据功能分子目录（如 `components/forms/`, `components/ui/`）

---

### `lib/`（工具函数与服务）

- 数据库调用、API 封装、验证逻辑、格式化函数等

✅ **最佳实践：**

- 每个服务模块一个文件，如 `auth.ts`、`stripe.ts`
    
- 避免和组件混杂在一起，保持职责清晰

---

### `styles/`（样式）

- 存放全局 CSS、Tailwind 配置或 CSS Module

✅ **最佳实践：**

- `globals.css`：放置 Tailwind、字体、主题等全局样式
    
- 使用 Tailwind 时避免冗余 class，可封装组件

---

### `public/`（静态资源）

- 不需导入的资源，如 logo、OG 图像、PDF 文件等

✅ **最佳实践：**

- 图片命名清晰有语义
    
- 分目录存放（如 `/images/`, `/icons/`）

---

### `middleware.ts`

- 中间件逻辑，如 auth 校验、重定向 

✅ **最佳实践：**

- 不要滥用 middleware，因其运行于 Edge runtime，性能敏感