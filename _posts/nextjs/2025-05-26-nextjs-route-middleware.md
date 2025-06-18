---
layout: post
title: Next.js入土计划05-请求服务和中间件介绍
subtitle: 全面解析 route.ts 与 middleware.ts，并结合完整 CRUD 示例与常用 next.config.ts 配置
author: Vascent
header-img: 
tags:
  - Nextjs
---


## `route.ts`
在 Next.js 的 App Router 架构中，`route.ts`（或 `route.js`）文件用于定义自定义的 API 路由处理器，允许你为特定路径创建自定义的请求处理逻辑。

### 🧭 `route.ts` 的作用
通过在 `app/` 目录下的特定路径中创建 `route.ts` 文件，你可以为该路径定义一个或多个 HTTP 方法的处理函数。这些函数使用 Web 的 Request 和 Response API 来处理请求和响应。
例如，这个路径结构: `app/api/hello/route.ts`, 将对应于 `/api/hello` 路径的 API 路由。

### 🛠️ 支持的 HTTP 方法

`route.ts` 文件中可以导出以下 HTTP 方法的处理函数：[Next.js+1Next.js+1](https://nextjs.org/docs/app/api-reference/file-conventions/route?utm_source=chatgpt.com) https://nextjs.org/docs/app/api-reference/file-conventions/route?utm_source=chatgpt.com

- `GET`
- `POST`
- `PUT`
- `PATCH`
- `DELETE`
- `HEAD`
- `OPTIONS`

### 📦 参数说明

#### `request`（可选）

处理函数的第一个参数是 `request` 对象，通常是 `NextRequest` 类型，它扩展了 Web 的 Request API，提供了更方便的访问方法，如：[Next.js](https://nextjs.org/docs/app/api-reference/file-conventions/route?utm_source=chatgpt.com)

- `request.nextUrl`：解析后的 URL 对象
- `request.cookies`: 访问请求中的 Cookie

```tsx
import { NextRequest } from 'next/server';

export async function GET(request: NextRequest) {
  const { searchParams } = request.nextUrl;
  const name = searchParams.get('name') || 'Guest';
  return new Response(`Hello, ${name}!`);
}
```

#### `context`（可选）

处理函数的第二个参数是 `context` 对象，包含路由参数等信息：[Next.js](https://nextjs.org/docs/app/api-reference/file-conventions/route?utm_source=chatgpt.com)

- `params`：一个包含动态路由参数的对象[Next.js+1Next.js+1](https://nextjs.org/docs/app/api-reference/file-conventions/route?utm_source=chatgpt.com)

```tsx
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  return new Response(`Item ID: ${params.id}`);
}
```
如果你的路由路径是 `app/api/items/[id]/route.ts`，当访问 `/api/items/123` 时，`params.id` 的值为 `'123'`。

## 🧩 与传统 API 路由的区别

在 Next.js 的 Pages Router 中，API 路由通常位于 `pages/api/` 目录下，并使用默认导出函数来处理请求。而在 App Router 中，`route.ts` 文件允许你为每个 HTTP 方法分别导出处理函数，提供了更清晰和模块化的方式来定义 API 路由。
1. **文件和目录结构 (Location & Convention):**
    - **Pages Router:**
        - 约定大于配置，API 必须放在 pages/api/ 目录下。
        - 文件名 (不含扩展名) 成为 API 路径的一部分。例如，pages/api/users.js 会映射到 /api/users。
        - 动态路由使用方括号，如 pages/api/users/[id].js。
		    https://nextjs.org/docs/pages/building-your-application/routing/api-routes
            
    - **App Router:**
        - 更灵活，API 路由 (route.ts 或 route.js 文件) 可以存在于 app 目录下的任何层级。例如，app/api/users/route.ts 或 app/products/[id]/api/route.ts。
        - 路径由目录结构决定，而 route.ts 是处理该路径请求的特殊文件。
        - 例如，app/api/items/route.ts 对应 /api/items 路径的 API。
        - app/items/[id]/route.ts 对应 /items/:id 路径的 API（注意，这里 api 目录不是必须的，可以直接在业务逻辑相关的目录旁创建 route.ts）。

2. **处理函数定义 (Handler Definition):**
- **Pages Router:**
```tsx
// pages/api/users.js
export default function handler(req, res) {
  if (req.method === 'GET') {
    // 处理 GET 请求
    res.status(200).json({ message: 'GET request to users' });
  } else if (req.method === 'POST') {
    // 处理 POST 请求
    res.status(200).json({ message: 'POST request to users' });
  } else {
    res.setHeader('Allow', ['GET', 'POST']);
    res.status(405).end(`Method ${req.method} Not Allowed`);
  }
}
```
这里，一个函数处理所有方法，需要内部逻辑来区分。
- **App Router:**
```tsx
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  // 处理 GET 请求
  return NextResponse.json({ message: 'GET request to users' });
}

export async function POST(request: NextRequest) {
  // 处理 POST 请求
  // const body = await request.json();
  return NextResponse.json({ message: 'POST request to users' });
}

// 如果没有导出某个方法 (如 PUT)，则该方法会返回 405 Method Not Allowed
```
每个 HTTP 方法对应一个独立的、具名的导出函数。Next.js 会自动将请求路由到相应的函数。

3. **请求和响应对象 (Request and Response Objects):**
	- **Pages Router:** 使用 Node.js 原生的 http.IncomingMessage (作为 req) 和 http.ServerResponse (作为 res) 对象，或者 Next.js 封装的 NextApiRequest 和 NextApiResponse。
	    
	- **App Router:** 全面拥抱 Web Standards。
    - 请求对象是标准的 Request 对象 (Next.js 扩展为 NextRequest)。
    - 处理函数必须返回一个标准的 Response 对象 (Next.js 提供了 NextResponse 辅助类)。


### 实践案例

#### 目录结构
```tsx
app/
└─ api/
   └─ items/
      ├─ route.ts        ← 列表查询 & 新增(Create, Read All)
      └─ [id]/
         └─ route.ts     ← 单条记录 Read One、Update、Delete
```

#### `app/api/items/route.ts`（Read All & Create）

```tsx
// app/api/items/route.ts
import { NextRequest, NextResponse } from 'next/server';

// 模拟内存中存储
export let items = [
  { id: '1', name: 'Item 1', description: 'Description 1' },
  { id: '2', name: 'Item 2', description: 'Description 2' },
];

// GET /api/items  — 返回所有 items
export async function GET() {
  return NextResponse.json({ data: items });
}

// POST /api/items — 新增一个 item
export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const newItem = {
      id: String(Date.now()),        // 简单用时间戳做 id
      name: body.name,
      description: body.description,
    };
    items.push(newItem);
    return NextResponse.json({ data: newItem }, { status: 201 });
  } catch (err) {
    return NextResponse.json({ error: 'Invalid JSON' }, { status: 400 });
  }
}
```


#### `app/api/items/[id]/route.ts`（Read One, Update, Delete）

```tsx
// app/api/items/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server';

// 引用同一个内存数组
// （在实际项目中应替换为数据库调用）
import { items } from '../route'; 

// GET /api/items/:id — 查询单条
export async function GET(
  _request: NextRequest,
  { params }: { params: { id: string } }
) {
  const item = items.find((i) => i.id === params.id);
  if (!item) {
    return NextResponse.json({ error: 'Not Found' }, { status: 404 });
  }
  return NextResponse.json({ data: item });
}

// PUT /api/items/:id — 更新
export async function PUT(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
 ...
}

// DELETE /api/items/:id — 删除
export async function DELETE(
  _request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  ...
}
```

> ⚠️ 这里是导入模块变量来模拟数据，修改文件时，需要重新启动服务。才能正常进行增删改查。

#### 在客户端组件中实现增删改查

```tsx
// app/components/todo-list.tsx
"use client";

import { Button } from "@/components/ui/button";
import { useEffect, useState } from "react";

const getItems = async () => {
  const res = await fetch('http://localhost:3000/api/items');
  const data = await res.json();
  return data.data;
};

const deleteItem = async (id: string) => {
  const res = await fetch(`http://localhost:3000/api/items/${id}`, {
    method: 'DELETE',
  });
  return res.json();
};

const addItem = async () => {
  const res = await fetch('http://localhost:3000/api/items', {
    method: 'POST',
    body: JSON.stringify({ name: 'New Item', description: 'New Item Description' }),
  });
  return res.json();
};

export default function TodoList() {
  const [data, setData] = useState([]);

  useEffect(() => {
    getItems().then(setData);
  }, []);
  ...
}
```

## `middleware.ts
在 **Next.js App Router** 中，`middleware.ts`（或 `middleware.js`）文件用于在每次请求到达路由之前，运行自定义的服务器端逻辑（如鉴权、重定向、日志埋点等），并可修改请求或响应。下面分项说明其用法：

### 1. 文件位置与命名
- https://nextjs.org/docs/app/api-reference/file-conventions/middleware
- 在项目根目录（与 `app/` 或 `pages/` 同级）下创建 `middleware.ts`（或 `.js`）。
- 如果你的源码放在 `src/` 中，也可置于 `src/middleware.ts`。

### 2. 必须导出的 Middleware 函数
```tsx
import { NextRequest, NextResponse } from 'next/server';

// 命名导出或 default 导出皆可
export function middleware(request: NextRequest) {
  // 基于请求 URL 做重定向
  if (request.nextUrl.pathname.startsWith('/admin')) {
    // 举例：未登录用户禁止访问 /admin
    const token = request.cookies.get('token');
    if (!token) {
      return NextResponse.redirect(new URL('/login', request.url));
    }
  }
  // 默认继续执行，返回 undefined 或 NextResponse.next()
  return NextResponse.next();
}
```
- **参数 `request`**：类型为 `NextRequest`，扩展了标准 Web 的 Request API，可访问 `request.nextUrl`、`request.cookies` 等。
    
- **返回值**：`NextResponse` 实例（或标准 `Response`），用于重写、重定向、修改 header、直接响应等；若返回 `undefined` 或 `NextResponse.next()`，则继续走后续路由渲染。

3. 可选的 `config` 对象：路径匹配器 (matcher)
```tsx
export const config = {
  // 针对单一路径
  matcher: '/about/:path*',
  // 或多个路径
  // matcher: ['/about', '/contact'],
  // 也支持复杂规则
  // matcher: ['/((?!api|_next/static|_next/image).*)'],
  /* 甚至可写对象数组，做更精细控制 */
  /* matcher: [
       {
         source: '/api/:path*',
         locale: false,
         has: [{ type: 'header', key: 'x-auth', value: 'yes' }],
       },
     ], */
};
```
- **简单字符串**：直接写路径或带通配符的 `/foo/:path*`。
- **数组**：列举多个路径。
- **正则 & 对象**：可指定 `source`、`regexp`、`locale`、`has`、`missing` 等，按请求头、查询参数、Cookie 存在/缺失等条件精确匹配。

### 4. `NextResponse` 的能力
- **重定向**：`NextResponse.redirect(new URL('/login', request.url))`
- **路径重写**：`NextResponse.rewrite(new URL('/api/proxy', request.url))`
- **修改 header**：
```tsx
const res = NextResponse.next();
res.headers.set('x-custom', 'value');
return res;
```
- **操作 Cookie**：`NextResponse.cookies.set('theme', 'dark')`

### 5. 运行时环境
- Middleware 默认在 **Edge Runtime** 下运行，性能更优、延迟更低。
- 如需在 **Node.js** 完整运行时运行（例如使用某些 Node-only 库），可在 `next.config.js` 中通过 `matcher` 或 `middleware` 配置指定 **运行时**（目前需配合实验性选项，详见官方升级指南）。


### 实践操作

#### **目录结构**

```cpp
my-next-app/
├─ app/
│  ├─ layout.tsx
│  ├─ page.tsx
│  ├─ admin/
│  │   └─ page.tsx
│  └─ login/
│      └─ page.tsx
├─ middleware.ts
├─ next.config.ts
└─ package.json
```

#### 编写`middleware.ts`
```tsx
// middleware.ts 在项目根目录下
import { NextRequest, NextResponse } from 'next/server';

// 只对 /admin 路径生效
export const config = {
  matcher: '/admin/:path*',
};

export function middleware(request: NextRequest) {
  // 简单日志：打印请求方法 & URL
  console.log(`[Middleware] ${request.method} ${request.nextUrl.pathname}`);

  // 鉴权示例：检查名为 "token" 的 Cookie
  const token = request.cookies.get('token')?.value;
  if (!token) {
    // 未登录：重定向到 /login，并携带原始请求路径
    const loginUrl = request.nextUrl.clone();
    loginUrl.pathname = '/login';
    loginUrl.searchParams.set('from', request.nextUrl.pathname);
    return NextResponse.redirect(loginUrl);
  }

  // 登录后：继续，并在响应头中加一个自定义标记
  const response = NextResponse.next();
  response.headers.set('X-User-Token', token);
  return response;
}
```

#### 管理页：`app/admin/page.tsx`

```tsx
// app/admin/page.tsx
export default function AdminPage() {
  return <h1>管理员面板 — 已通过中间件鉴权 🔒</h1>;
}
```

#### 登录页：`app/login/page.tsx`

```tsx
// app/login/page.tsx
'use client';
import { useSearchParams, useRouter } from 'next/navigation';

export default function LoginPage() {
  const params = useSearchParams();
  const from = params.get('from') || '/';
  const router = useRouter();

  const handleLogin = () => {
    // 模拟登录：写入名为 "token" 的 Cookie
    document.cookie = 'token=demo-token; path=/';
    // 登录后重定向回原始页面
    router.push(from);
  };

  return (
    <div>
      <h1>请先登录 🔐</h1>
      <button onClick={handleLogin} style={{ padding: '8px 16px', marginTop: 16 }}>
        模拟登录
      </button>
    </div>
  );
}
```

访问 **`http://localhost:3000/admin`**
- 若浏览器没有 `token` Cookie，会被重定向到 `http://localhost:3000/login?from=/admin`；
-    服务端控制台打印：`[Middleware] GET /admin`
- 在登录页点击「模拟登录」
	- 会在浏览器写入 `token=demo-token`，并自动跳回 `/admin`；
	- 中间件再次运行，打印日志，并在响应头可见 `X-User-Token: demo-token`；
	- 最终展示 Admin 页面内容。
- 访问首页或其他路径（如 `/`、`/login`）
	- 中间件不生效，不会拦截。

## **Next.js常用配置**

```ts
/**
 * Next.js 常用配置示例 (适用于最新版本, App Router + TypeScript)
 * - 启用 App Router (appDir)
 * - 图片优化 (Image domains, formats, loader)
 * - 国际化 (i18n)
 * - Edge 路由支持 (通过在路由文件中设置 runtime = 'edge')
 * - SWR 缓存控制 (Cache-Control 头, revalidate)
 * - AI 路由支持 (通过 Vercel AI SDK 进行流式响应)
 * - 实验性功能标注 (如 Turbopack, View Transitions)
 */

import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  // React 严格模式 (强烈推荐在开发环境启用)
  reactStrictMode: true,

  // 图片优化配置
  images: {
    // 允许优化的外部主机名列表
    domains: ['example.com', 'images.example.com'],
    // 支持的现代图片格式 (当浏览器支持时优先使用 AVIF/WebP)
    formats: ['image/avif', 'image/webp'],
    // 图片加载器类型 (默认为 'default' 使用 Next.js 内置优化)
    loader: 'default',
    // 使用 remotePatterns 来匹配更复杂的外部 URL（示例：加载特定 CDN 路径下的图片）
    /*
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'cdn.example.com',
        pathname: '/images/**',
      },
    ],
    */
    // 图片缓存的最低过期时间 (秒)
    minimumCacheTTL: 60 * 60 * 24, // 24 小时
  },

  // 国际化 (i18n) 配置
  i18n: {
    // 支持的语言列表
    locales: ['en', 'zh-CN', 'es-ES'],
    // 默认语言
    defaultLocale: 'en',
    // 可选：不同域名对应的默认语言（用于多域名部署）
    domains: [
      {
        domain: 'example.com',
        defaultLocale: 'en',
      },
      {
        domain: 'example.cn',
        defaultLocale: 'zh-CN',
      },
    ],
  },

  // 自定义 HTTP 头（可用于控制 SWR 缓存策略）
  async headers() {
    return [
      {
        // 为所有页面设置缓存头（示例：启用 stale-while-revalidate）
        source: '/(.*)',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=0, s-maxage=60, stale-while-revalidate=300',
          },
        ],
      },
      {
        // 对于 API 路径可设置不同策略（示例：API 默认不缓存）
        source: '/api/(.*)',
        headers: [
          {
            key: 'Cache-Control',
            value: 'no-store',
          },
        ],
      },
    ];
  },

  // 实验性及高级配置
  experimental: {
    // 启用 App Router (appDir) - Next.js 13.4+ 默认为 true，可不显式设置
    appDir: true,

    // Turbopack (实验性) - Next.js 15 默认支持，可配置别名等（仅开发模式生效）
    // turbo: {
    //   resolveAlias: {
    //     underscore: 'lodash',
    //   },
    // },

    // React 视图过渡 (实验性) - 启用后可使用 React View Transitions API（不建议生产环境使用）
    viewTransition: true,

    // AI 路由支持：Next.js 可在 Edge 环境中流式处理 AI 请求（使用 Vercel AI SDK 等）
    // 无需额外 next.config 配置；在 route.ts 中使用 `export const runtime = 'edge'` 并调用 AI SDK
  },

  // 以下为 Next.js 14/15 新增的常见优化项，可按需启用
  // trailingSlash: true, // 在所有路由后添加斜杠
  // productionBrowserSourceMaps: true, // 生产环境启用 Source Map
  // eslint: {
  //   ignoreDuringBuilds: true, // 构建时忽略 ESLint 错误
  // },
};

export default nextConfig;
```

Demo: https://github.com/Yangeyu/nextjs-tutorial/tree/main/lesson-06-route-middleware

## 节外生枝
**StyleGlide Themes** 是一个基于 AI 的“设计系统模板市场”，主要功能包括：
1. **生成配色方案（Color Palettes）**
2. **生成排版样式（Typography Styles）**
3. **一键下载并分发给 shadcn/UI 等组件库**

![Kapture 2025-05-26 at 12.00.32.gif](https://raw.githubusercontent.com/Yangeyu/img-hoist/main/Kapture%202025-05-26%20at%2012.00.32.gif)

https://www.styleglide.ai/themes




