---
layout: post
title: "tsconfig - declaration & outDir & declarationDir"
subtitle: "介绍 TS 类型文件输出路径的配置"
author: "Vascent"
header-img: "//w.wallhaven.cc/full/yx/wallhaven-yxqlzg.png"
<!-- header-style: text -->
tags:
  - tsconfig.json
  - TypeScript
---

# tsconfig - declaration & outDir & declarationDir

### declaration

`declaration` 配置为 `true` 时。将为项目中的每个 TypeScript 或 JavaScript 文件生成 `d.ts` 文件。这些 `d.ts` 文件是描述模块外部 API 的类型定义文件。

***Expamle***

```js
// tsconfig.json
{
  declaration: true
}
```

```tsx
// index.ts
export let helloWorld = "hi";
```

经过编译后会生成

```tsx
//index.js
export let helloWorld = "hi";

// index.d.ts
export declare let helloWorld: string;
```

| 💡 如果只想编译生成 `.d.ts` 文件，可以新增配置:  [`emitDeclarationOnly: true`](https://www.typescriptlang.org/tsconfig#emitDeclarationOnly)

### declarationDir

配置 `d.ts` 文件输出目录

***Example***

```
example
├── index.ts
├── package.json
└── tsconfig.json
```

```js
// tsconfig.json
{
  "compilerOptions": {
    "declaration": true,
    "declarationDir": "./types"
  }
}
```

编译后将在 `index.ts` 文件的目录下新增 `types/index.d.ts`。

```
example
├── index.js
├── index.ts
├── package.json
├── tsconfig.json
└── types
    └── index.d.ts
```

### ourDir

这个配置会指定 `.js` (还有 `.d.ts`， `.js.map`，等等)的输出路径。如果未指定值，则编译后生成的文件会放置于 `.ts` 文件所在目录下。

***Example*** 

```
src
├── example
│   └── index.ts
└── index.ts
```

- 未配置 `outDir` 经过编译后
    
    ```
    src
    ├── example
    │   ├── index.d.ts
    │   ├── index.js
    │   └── index.ts
    ├── index.d.ts
    ├── index.js
    └── index.ts
    ```
    
- 配置 `outDir`
    
    ```json
    {
      "compilerOptions": {
        "outDir": "src/dist",
    		"declaration": true,
      }
    }
    ```
    
    编译后的工程目录
    
    ```
    src
    ├── dist
    │   ├── example
    │   │   ├── index.d.ts
    │   │   └── index.js
    │   ├── index.d.ts
    │   └── index.js
    ├── example
    │   └── index.ts
    └── index.ts
    ```
    
- 同时配置 `outDir` 和 `declarationDir`，类型文件的输出路径根据 `declarationDir`。
    
    ```json
    {
      "compilerOptions": {
        "outDir": "src/dist",
    		"declaration": true,
    		"declarationDir": "src/dist/types/", 
    }
    ```
    
    ```
    src
    ├── dist
    │   ├── example
    │   │   └── index.js
    │   ├── types
    │   │   ├── example
    │   │   │   └── index.d.ts
    │   │   └── index.d.ts
    │   └── index.js
    ├── example
    │   └── index.ts
    └── index.ts
    ```
