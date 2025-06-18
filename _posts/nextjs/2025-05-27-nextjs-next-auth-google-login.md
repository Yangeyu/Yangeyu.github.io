---
layout: post
title: Next.js入土计划06-基于NextAuth实现谷歌邮箱登陆
author: Vascent
header-img: 
tags:
  - Nextjs
---


## 结合 **next-auth** 实现Google登陆

使用 OAuth 2.0 访问 Google API配置教程：
https://apifox.com/apiskills/how-to-use-google-oauth2/
在"Authorized redirect URIs"配置项中添加：http://localhost:3000/api/auth/callback/google
![image.png](https://raw.githubusercontent.com/Yangeyu/img-hoist/main/20250527102708887.png)


### 安装依赖

```shell
pnpm add next-auth
```

### 环境变量

```tsx
GOOGLE_CLIENT_ID=你的GoogleClientID
GOOGLE_CLIENT_SECRET=你的GoogleClientSecret
NEXTAUTH_SECRET=任意一串强随机字符串
NEXTAUTH_URL=http://localhost:3000
```

### 创建Auth 配置文件
```tsx
// lib/auth.ts
import { NextAuthOptions } from "next-auth";
import CredentialsProvider from "next-auth/providers/credentials";
import GoogleProvider from "next-auth/providers/google";

// 扩展 NextAuth 的类型定义
declare module "next-auth" {
  // 扩展 Session 接口，添加自定义用户属性
  interface Session {
    user: {
      id?: string;       // 用户ID
      name?: string | null;  // 用户名称
      email?: string | null; // 用户邮箱
      image?: string | null; // 用户头像
    }
  }
}

// NextAuth 配置选项
export const authOptions: NextAuthOptions = {
  // 配置身份验证提供者
  providers: [
    // Google OAuth 提供者配置
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID as string,      //  Google OAuth 应用的客户端ID
      clientSecret: process.env.GOOGLE_CLIENT_SECRET as string, // Google OAuth 应用的客户端密钥
    }),
    
    // 凭证(用户名密码)提供者配置
    CredentialsProvider({
      name: "Credentials", // 提供者名称
      // 定义登录表单字段
      credentials: {
        username: { label: "Username", type: "text", placeholder: "username" }, // 用户名输入框
        password: { label: "Password", type: "password" },                     // 密码输入框
      },
      // 验证用户凭证的函数
      async authorize(credentials) {
        // 这里通常会检查数据库中的用户凭证
        // 示例中使用硬编码的用户名和密码进行演示
        if (credentials?.username === "admin" && credentials?.password === "password") {
          // 验证成功，返回用户对象
          return { id: "1", name: "Admin", email: "admin@example.com" };
        }
        // 验证失败，返回null
        return null;
      },
    }),
  ],
  
  // 自定义页面路径
  pages: {
    signIn: "/login", // 自定义登录页面路径
  },
  
  // 回调函数配置
  callbacks: {
    // session 回调：在每次会话被访问时调用
    async session({ session, token }) {
      // 如果存在令牌和会话用户信息
      if (token && session.user) {
        // 将令牌中的用户ID添加到会话用户对象中
        session.user = {
          ...session.user,
          id: token.sub as string // 使用令牌的sub字段作为用户ID
        };
      }
      // 返回更新后的会话
      return session;
    },
  },
  
  // 会话配置
  session: {
    strategy: "jwt", // 使用JWT策略管理会话
  },
};
```

在 `app/api/auth/[...nextauth]/route.ts` 中：

```tsx
import { authOptions } from "@/lib/auth";
import NextAuth from "next-auth/next";

const handler = NextAuth(authOptions);

export { handler as GET, handler as POST };
```

### 全局布局：`app/layout.tsx`

在根布局中包裹 `SessionProvider`：
```tsx
// components/AuthProvider.tsx
"use client";

import { SessionProvider } from "next-auth/react";
import { ReactNode } from "react";

interface AuthProviderProps {
  children: ReactNode;
}

export default function AuthProvider({ children }: AuthProviderProps) {
  return <SessionProvider>{children}</SessionProvider>;
}

```

```tsx
// app/layout.tsx
import AuthProvider from "@/components/AuthProvider";
import Navbar from "@/components/Navbar";

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
        <AuthProvider>
          <Navbar />
          <div className="min-h-screen bg-gray-50">
            {children}
          </div>
        </AuthProvider>
  );
}

```

### 编写一个客户端的导航栏(可选)
```tsx
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";
import { useSession, signOut } from "next-auth/react";
import { Button } from "@/components/ui/button";

export default function Navbar() {
  const pathname = usePathname();
  const { data: session, status } = useSession();
  const loading = status === "loading";
...
```

#### `useSession`
`useSession` 是一个 **React Hook**，只能在 **客户端组件**（带有 `"use client"` 的组件）中调用。

**返回值结构**  
`useSession()` 返回一个对象，常用的有两个字段：

1. **`data`**（这里重命名为 `session`）
    - 类型：`Session \| null`
    - 如果用户已登录，`session` 是一个对象，包含用户信息（如 `session.user.name`、`session.user.email`、`session.user.image` 等）。
    - 如果未登录或会话过期，则为 `null`。
        
2. **`status`**
    - 类型：`"loading" | "authenticated" | "unauthenticated"`
    - `"loading"`：正在确定会话状态（通常在首次渲染时或页面刷新时）。
    - `"authenticated"`：已确认用户已登录。
    - `"unauthenticated"`：用户未登录。

#### `getServerSession`
相对的 `getServerSession` 是 NextAuth.js 提供的一个 **服务端** 函数，用于在 **Server Component**、**`getServerSideProps`**、**API Route** 或 **Route Handler** 中，**同步** 获取当前请求的用户会话数据。它的主要作用和特点如下：
- **在服务器端获取会话**  
    直接从请求的 Cookie 或 Authorization 头中解析并验证会话令牌，返回当前用户的 `Session` 对象或 `null`。
- **支持 SSR/SSG 阶段**  
    可以在 Next.js 的 **Server Component**（`app/` 目录）或 **`getServerSideProps`**（`pages/` 目录）中调用，用来做鉴权、数据注入或页面重定向。
- 	**函数签名**
```tsx
import { getServerSession } from "next-auth/next"
import type { NextAuthOptions } from "next-auth"

// authOptions 与你在 [...nextauth]/route.ts 中 export const authOptions 保持一致
export async function getServerSession(
  req: NextRequest | GetServerSidePropsContext["req"],
  res: NextResponse | GetServerSidePropsContext["res"],
  authOptions: NextAuthOptions
): Promise<Session | null>
```

- **`req`**：来自 Next.js 的请求对象
    - 在 App Router 下为 `NextRequest`
    - 在 Pages Router 下为 Node.js 原生 `IncomingMessage` 或 `GetServerSidePropsContext["req"]`
        
- **`res`**：来自 Next.js 的响应对象
    - 在 App Router 下可传入 `undefined`（内部会自动处理）
    - 在 Pages Router 下为 Node.js 原生 `ServerResponse` 或 `GetServerSidePropsContext["res"]`
        
- **`authOptions`**：与你在 `route.ts` 中配置的 NextAuth 选项（`providers`、`adapter`、`secret` 等）一致。
- **返回值**：
    - 成功时：`Session` 对象，包含 `user` 信息（`name`、`email`、`image` 等）和 `expires` 时间。
    - 无登录或会话过期时：`null`。

```tsx
// app/page.tsx
import { Button } from "@/components/ui/button";
import { getServerSession } from "next-auth/next";
import { authOptions } from "@/lib/auth";
import Link from "next/link";

export default async function Home() {
  const session = await getServerSession(authOptions);
  
  return (
    <div className="flex flex-col gap-4 items-center">
      <main className="p-4 max-w-4xl mx-auto text-center py-12">
        
        <div className="flex gap-4 justify-center">
          {session ? (
            <>
              <Button asChild>
                <Link href="/dashboard">Go to Dashboard</Link>
              </Button>
              <SignOutButton />
            </>
          ) : (
            <Button asChild>
              <Link href="/login">Sign In</Link>
            </Button>
          )}
        </div>
      </main>
....
    </div>
  );
}

```


|特性|`getServerSession`（服务端）|`useSession`（客户端）|
|---|---|---|
|执行环境|服务端（Node.js/Edge）|客户端（浏览器）|
|用法|同步／异步函数调用|React Hook|
|返回时机|渲染前、请求处理阶段|首次渲染后|
|重渲染或订阅|不订阅，仅执行一次|自动订阅会话变化|
|典型场景|SSR、API 鉴权、Server Component|页面内条件渲染、UI 逻辑|
