---
layout: post
title: Next.js入土计划07-Nextjs结合Prisma实现登录注册
---

## Prisma

Prisma 是一个现代的数据库工具，它通过类型安全和自动生成的数据库客户端，帮助开发者更高效、可靠地与数据库交互。适用于 Node.js 和 TypeScript 项目，广泛用于构建基于数据库的应用。

### 1. 安装依赖

```shell
npm install prisma --save-dev
npm install @prisma/client
```

初始化 Prisma 配置：
```shell
npx prisma init
```

这会生成一个 `prisma/` 文件夹，里面包含：
- `schema.prisma`：数据库 schema 定义文件
- `.env`：配置数据库连接字符串的环境变量

### 2. 配置数据库连接
编辑 `.env` 文件，填写你的数据库地址：
```tsx
DATABASE_URL="postgresql://user:password@localhost:5432/mydb"
```

### 3. 定义数据模型（schema.prisma）
例如：
```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  posts     Post[]
  createdAt DateTime @default(now())
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User?    @relation(fields: [authorId], references: [id])
  authorId  Int?
}

```

### 4. 生成数据库迁移 & 创建表

```shell
npx prisma migrate dev --name init
```

这会：
- 创建并应用数据库迁移
- 自动生成 Prisma Client 代码（用于数据库操作）

Prisma 会在开发环境下完成以下几步：
1. 加载环境变量与 Schema
	- **`.env`**  
	    Prisma 会优先从项目根目录的 `.env` 文件中读取 `DATABASE_URL`（以及可选的 `SHADOW_DATABASE_URL`）
	- **`schema.prisma`**  
    读取 `prisma/schema.prisma` 中的数据源（`datasource`）与模型定义（`model`）

2. 计算迁移差异（Diff）
Prisma 会对 **当前的数据库结构** 与 **`schema.prisma` 中定义的模型** 做一次「差异比对」：
	1. 如果这是第一次运行迁移，则视为空数据库 → 会基于所有 `model` 生成针对空库的建表 SQL
	2. 如果已有历史迁移，则对比最新迁移后的数据库结构与新的模型定义 → 生成追加或修改 SQL

3. 生成迁移脚本文件
- Prisma 会在项目的 `prisma/migrations/` 目录下生成一个以时间戳开头、你给定名字（`init`）结尾的子目录，类似这样：

```
prisma/
└─ migrations/
   └─ 20250527153045_init/
      ├─ migration.sql
      ├─ schema.prisma   ←（可选）快照
      └─ README.md       ←（可选）描述
```
这个子目录的作用主要有以下几点：
1. **版本化管理**  
    每个子目录代表一次迁移（migration），名称以 UTC 时间戳保证全局唯一，并按创建先后排序。你可以把整个 `prisma/migrations/` 目录纳入版本控制（Git），这样团队内每个人在不同时间生成的迁移都能被记录、追踪、回滚。
2. **存储迁移 SQL**  
    `migration.sql` 文件里就是 Prisma 根据 `schema.prisma` 与当前数据库状态的「差异」自动生成的 DDL（Data Definition Language）脚本。它包含了所有新增表、修改字段、添加索引等操作，且可以被手动审查、调整甚至补充自定义 SQL。
3. **驱动迁移流程**  
    当你运行 `prisma migrate dev`、`prisma migrate deploy` 或 `prisma migrate reset` 等命令时，Prisma 会读取 `prisma/migrations/` 下各个子目录里的 `migration.sql`，按目录名排序依次执行，确保你的数据库总是按照正确的顺序、逐步地应用所有结构变更。
4. **保持 Schema 快照（可选）**  
    在较新版本中，Prisma 还会在子目录里生成一个 `schema.prisma` 的快照，以记录当时生成这次迁移所依据的模型定义。这样你就能随时对比某次迁移前后 `schema.prisma` 的差别，提升可维护性。
5. **自动化与回滚**  
    Prisma 在数据库中维护一个 `_prisma_migrations` 表，记录每次成功应用过的迁移目录名。后续命令会据此决定哪些迁移已经执行过，哪些还需要执行，也能用来做回滚测试或重置操作。

子目录里会包含一个 `migration.sql`，内容就是第 2 步计算出的 DDL 语句，比如：
```
-- DropTable, CreateTable, AlterTable ...
CREATE TABLE "User" (
  "id" SERIAL PRIMARY KEY,
  "email" TEXT NOT NULL UNIQUE,
  "name" TEXT,
  "createdAt" TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE TABLE "Post" ( ... );
```

4. 应用迁移到数据库
Prisma 会依次执行生成的 SQL 脚本，让你的数据库真实地**创建**／**修改**表结构：
- 在开发模式 `migrate dev` 下，它还会使用一个 **shadow database**（除非你指定了直连且已构建），来预演迁移，确保不会在实际数据库上出错
- 执行后的结果，Prisma 会在数据库中维护一个 `_prisma_migrations` 表，用来记录哪些迁移已经应用

5. 自动生成（或更新）Prisma Client

一旦迁移成功，Prisma 紧接着会调用：
```shell
prisma generate
```

- 根据最新的 `schema.prisma`，在 `node_modules/.prisma/client/`（以及你项目中 `@prisma/client` 包）下生成 **类型安全的、与你模型一一对应的 Client API**
- 你在代码中引入的：
```tsx
import { PrismaClient } from "@prisma/client";
```
会立即对应到最新的 TypeScript 类型与方法（比如：`prisma.user.findMany`、`prisma.post.create` 等）

### 5. 使用 Prisma Client 查询数据
在项目中导入并使用 Prisma Client：
```tsx
// lib/prisma.ts
import { PrismaClient } from '@prisma/client'
export const prisma = new PrismaClient()
```

```tsx
// 使用示例
const users = await prisma.user.findMany()
```

常用操作：
```tsx
// 创建
await prisma.user.create({
  data: { email: 'test@example.com', name: 'Tom' },
})

// 查询
const user = await prisma.user.findUnique({
  where: { email: 'test@example.com' },
})

// 更新
await prisma.user.update({
  where: { email: 'test@example.com' },
  data: { name: 'Updated Name' },
})

// 删除
await prisma.user.delete({
  where: { email: 'test@example.com' },
})
```

### 6. 使用 Prisma Studio 查看数据

```shell
npx prisma studio
```

会打开一个本地 Web 页面，让你可视化查看和编辑数据库中的内容。

## NextAuth 整合集成数据库(Postgresql)

配置文档：https://authjs.dev/getting-started/adapters/prisma?_gl=1*n26gw4*_gcl_au*NDI1MDk3ODA5LjE3NDgyNTE4NTU.
可以通过neon创建一个数据库： https://console.neon.tech/app/projects

1. 在 `prisma/schema.prisma` 中描述数据模型：

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// 用户模型
model User {
  id            String    @id @default(cuid())
  name          String?
  email         String?   @unique
  emailVerified DateTime?
  image         String?
  password      String?
  accounts      Account[]
  sessions      Session[]
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
}
...
```
流程步骤参考上面prisma

2. 在next-auth配置中结合prisma实现用户校验逻辑

```tsx
// lib/auth.ts
import { NextAuthOptions } from "next-auth";
import CredentialsProvider from "next-auth/providers/credentials";
import prisma from "./prisma";
import { PrismaAdapter } from "@auth/prisma-adapter";
import { compare } from "bcrypt";

// NextAuth 配置选项
export const authOptions: NextAuthOptions = {
  // 配置身份验证提供者
  adapter: PrismaAdapter(prisma),
  providers: [
    // 凭证(用户名密码)提供者配置
    CredentialsProvider({
      name: "Credentials", // 提供者名称
      // 验证用户凭证的函数
      async authorize(credentials) {
        ...
        try {
          // 查找用户
          const user = await prisma.user.findUnique({
            where: {
              email: credentials.username,
            },
          });

          // 验证密码
          const isPasswordValid = await compare(credentials.password, user.password);
		...
        }
      },
    }),
  ],
  ...
};

```

3. 在 `lib/prisma.ts` 中创建一个跨热重载安全的单例：
```tsx
import { PrismaClient } from "@prisma/client";

// PrismaClient 是一个较重的实例，不应在每次请求时都创建
// 这里创建一个全局单例以便在整个应用中重用
const globalForPrisma = global as unknown as { prisma: PrismaClient };

// 检查是否已经存在 Prisma 实例，没有则创建新实例
export const prisma = globalForPrisma.prisma || new PrismaClient();

// 在开发环境外，将 Prisma 实例添加到全局对象中以避免热重载时创建多个连接
if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;

export default prisma;
```

4. 实现用户注册的接口逻辑
```tsx
// app/api/register/route.ts
import { NextResponse } from "next/server";
import prisma from "@/lib/prisma";
import { hash } from "bcrypt";

export async function POST(req: Request) {
  try {
    // 创建新用户
    const user = await prisma.user.create({
      data: {
        name,
        email,
        password: hashedPassword,
      },
    });
...
  } 
} 
```

5. 登陆的调用逻辑
账号密码登录：
```tsx
import { signIn } from "next-auth/react";

// 账号密码登录
const result = await signIn("credentials", {
	username,
	password,
	redirect: false,
});
  ...
```

第三方登录：
```tsx
import { getProviders, signIn } from "next-auth/react";
// 通过getProviders 获取providerid
const providers = await getProviders();
...
// GOOGLE 登陆
signIn(provider.id, { callbackUrl: "/" }
```

账号注册
```tsx
  ...
  const response = await fetch("/api/register", {
	method: "POST",
	headers: { "Content-Type": "application/json" },
	body: JSON.stringify({
	  name: formData.name,
	  email: formData.email,
	  password: formData.password,
	}),
  });
  ...

```

`getProviders()` 
- **作用**  
    `getProviders()` 用于获取当前 NextAuth.js 配置中所有可用的登录（认证）提供商（Provider）列表。
- **调用位置**
    - **客户端**（Client Side）：可在 React 组件或任何浏览器环境中调用
    - **服务端**（Server Side）：可在 API Route、`getServerSideProps`、Edge Function 等地方直接调用
- **返回值**
	- 调用后会返回一个对象（`Record<string, ClientSafeProvider>`），其中每个 key 都是 provider 的 `id`，value 是该 provider 的配置信息，例如：
```tsx
type ClientSafeProvider = Omit<Provider, "options"> & {
  signinUrl: string
  callbackUrl: string
}

interface Provider {
  id: string
  name: string
  type: "oauth" | "email" | "credentials"
  signinUrl: string
  callbackUrl: string
  // … 其他字段
}
```
你可以拿到每个 provider 的 `id`、`name`、`signinUrl`（发起登录跳转的 URL）、`callbackUrl` 等，进而动态渲染自定义登录按钮或链接。

`signIn()`
- **定义**：`signIn()` 是 NextAuth.js 在客户端（浏览器环境）提供的一个方法，用于触发用户登录流程。调用后会自动处理重定向和 CSRF Token（对于邮箱登录），并在登录完成后把用户带回原始发起登录的页面。
- **调用位置**：只能在客户端代码中使用（例如 React 组件或 `useEffect` 中）；在服务端（如 API Route）不可使用。

**基本用法**
1. 默认跳转到内置登录页
```tsx
import { signIn } from "next-auth/react"

export default function SignInButton() {
  return <button onClick={() => signIn()}>Sign in</button>
}
```
**效果**：调用 `signIn()`（不带参数）时，用户会被重定向到 NextAuth.js 提供的默认 `/api/auth/signin` 登录页。
 
2. 直接启动 OAuth 流程
```tsx
import { signIn } from "next-auth/react"

export default function GoogleSignIn() {
  return <button onClick={() => signIn("google")}>Sign in with Google</button>
}
```
**效果**：传入某个已配置的 Provider `id`（例如 `"google"`、`"github"`），会跳过中间的选择页，直接发起该 OAuth 提供商的授权流程。

3. 启动邮箱登录流程
```tsx
import { signIn } from "next-auth/react"

export default function EmailSignIn({ email }: { email: string }) {
  return <button onClick={() => signIn("email", { email })}>Sign in with Email</button>
}
```
**效果**：对于通过邮箱验证（magic link）方式登录，需将 `provider id` 设为 `"email"`，并在第二个参数中传入目标邮箱 `{ email: string }`，NextAuth.js 会自动处理发送邮件及 CSRF 安全检查。

**可选参数详解**
```tsx
signIn(
  providerId?: string,            // 可选，指定某个 provider（OAuth 或 email）；不传则先展示登录页
  options?: {
    callbackUrl?: string,         // 可选，登录成功后重定向的 URL（默认为当前页面 URL）
    redirect?: boolean,           // 可选，仅对 credentials/email 有效；false 表示不自动重定向，直接获取返回结果
    // 以及 provider 特有的字段，如 email、password 等
  }
)
```

`signOut()`
- **作用**  
    用于结束当前用户会话（登出），并自动处理 CSRF Token。
- **调用位置**  
    仅能在客户端（浏览器环境）调用，如 React 组件中的事件处理函数。
- **重定向行为**  
    默认会在登出完成后重新加载当前页面，保证用户回到发起登出时所在的页面。
    
**基本用法**
```tsx
import { signOut } from "next-auth/react";

export default function SignOutButton() {
  return (
    <button onClick={() => signOut()}>
      登出
    </button>
  );
}
```
点击按钮后，NextAuth.js 会发送登出请求到 `/api/auth/signout`，在成功删除会话后，浏览器会重新加载当前页面。

**可选参数**
1. `callbackUrl`
- **用途**：指定登出后希望跳转到的目标 URL。
- **使用方式**：将 URL 作为选项传给 `signOut()`。

2. `redirect: false`
- **用途**：在不触发页面刷新的情况下完成登出，仅删除会话并更新客户端状态。
- **返回值**：一个 `Promise<{ url: string }>`，其中 `url` 是校验后的 `callbackUrl`（或默认重定向地址）。


Demo: https://github.com/Yangeyu/nextjs-tutorial/tree/main/lesson-07-next-auth-goole-login