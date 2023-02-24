---
layout: post
title: "tsconfig.json - moduleResolution"
subtitle: '介绍moduleResolution配置'
author: "Vascent"
header-img: "//images.unsplash.com/photo-1528459801416-a9e53bbf4e17?ixlib=rb-4.0.3&q=80&fm=jpg&crop=entropy&cs=tinysrgb"
<!-- header-style: text -->
tags:
  - tsconfig.json
  - TypeScript
---

# tsconfig.json - moduleResolution

## Overview

| 💡 *模块解析：*是指编译器在查找导入模块内容时所遵循的流程
| 💡 *两种策略：*[Node](https://www.tslang.cn/docs/handbook/module-resolution.html#node) 和 [Classic](https://www.tslang.cn/docs/handbook/module-resolution.html#classic)

可以使用 `--moduleResolution` 标记来指定使用哪种模块解析策略。若未指定，那么在使用了

 `--module AMD | System | ES2015` 时的默认值为 [Classic](https://www.tslang.cn/docs/handbook/module-resolution.html#classic) ，其它情况时则为 [Node](https://www.tslang.cn/docs/handbook/module-resolution.html#node) 。

## Classic 策略

---

### 相对模块导入

| 💡 相对导入的模块是相对于导入它的文件进行解析的

**Example: `/root/src/folder/A.ts`**

```bash
import { b } from "./moduleB"
```

- ***查找流程***：
    1. `/root/src/folder/moduleB.ts`
    2. `/root/src/folder/moduleB.d.ts`

### 非相对模块的导入

| 💡 编译器则会从包含导入文件的目录开始依次向上级目录遍历，尝试定位匹配的声明文件

**Example: `/root/src/folder/A.ts`**

```bash
import { b } from "moduleB"
```

- ***查找流程***
    1. `/root/src/folder/moduleB.ts`
    2. `/root/src/folder/moduleB.d.ts`
    3. `/root/src/moduleB.ts`
    4. `/root/src/moduleB.d.ts`
    5. `/root/moduleB.ts`
    6. `/root/moduleB.d.ts`
    7. `/moduleB.ts`
    8. `/moduleB.d.ts`

## Node 策略

| 💡 这个解析策略试图在运行时模仿[Node.js](https://nodejs.org/)模块解析机制


### Node.js解析模块

---

### 相对模块解析

**Example: `/root/src/moduleA.js`**

```bash
var x = require("./moduleB");
```

- ***查找流程***
    1. 检查`/root/src/moduleB.js`文件是否存在。
    2. 检查`/root/src/moduleB`目录是否包含一个`package.json`文件，且`package.json`文件指定了一个`"main"`模块。
        - `/root/src/moduleB/package.json` 包含 `{ "main": "lib/mainModule.js" }`
        - Node.js会引用`/root/src/moduleB/lib/mainModule.js`。
    3. 检查`/root/src/moduleB`目录是否包含一个`index.js`文件。 这个文件会被隐式地当作那个文件夹下的"main"模块。

### 非相对模块解析

**Example: `/root/src/moduleA.js`**

```bash
var x = require("moduleB");
```

- ***查找流程***
    1. `/root/src/node_modules/moduleB.js`
    2. `/root/src/node_modules/moduleB/package.json` (如果指定了`"main"`属性)
    3. `/root/src/node_modules/moduleB/index.js`
    
    ---
    
    1. `/root/node_modules/moduleB.js`
    2. `/root/node_modules/moduleB/package.json` (如果指定了`"main"`属性)
    3. `/root/node_modules/moduleB/index.js`
    
    ---
    
    1. `/node_modules/moduleB.js`
    2. `/node_modules/moduleB/package.json` (如果指定了`"main"`属性)
    3. `/node_modules/moduleB/index.js`
    
    ❗ Node.js在步骤（4）和（7）会向上跳一级目录。
    
    
| 💡 Node.js先查找`moduleB.js`文件，然后是合适的`package.json`，再之后是`index.js`

### TypeScript解析模块

*TypeScript是模仿Node.js运行时的解析策略来在编译阶段定位模块定义文件*。 因此TypeScript在Node解析逻辑基础上增加了

- TypeScript源文件的扩展名( `.ts` , `.tsx`和`.d.ts` )。
- TypeScript在 `package.json` 里使用字段`"types"` 来表示类似`"main"` 的意义 - 编译器会使用它来找到要使用的"main"定义文件。

---

### 相对模块导入

**Example: `/root/src/moduleA.js`**

```bash
import { b } from "./moduleB"
```

- ***查找流程***
    1. `/root/src/moduleB.ts`
    2. `/root/src/moduleB.tsx`
    3. `/root/src/moduleB.d.ts`
    4. `/root/src/moduleB/package.json` (如果指定了`"types"`属性)
    5. `/root/src/moduleB/index.ts`
    6. `/root/src/moduleB/index.tsx`
    7. `/root/src/moduleB/index.d.ts`

### 非相对模块解析

**Example: `/root/src/moduleA.js`**

```bash
var x = require("moduleB");
```

- ***查找流程***
    1. `/root/src/node_modules/moduleB.ts`
    2. `/root/src/node_modules/moduleB.tsx`
    3. `/root/src/node_modules/moduleB.d.ts`
    4. `/root/src/node_modules/moduleB/package.json` (如果指定了`"types"`属性)
    5. `/root/src/node_modules/moduleB/index.ts`
    6. `/root/src/node_modules/moduleB/index.tsx`
    7. `/root/src/node_modules/moduleB/index.d.ts`
    
    ---
    
    1.  `/root/node_modules/moduleB.ts`
    2. `/root/node_modules/moduleB.tsx`
    3. `/root/node_modules/moduleB.d.ts`
    4. `/root/node_modules/moduleB/package.json` (如果指定了`"types"`属性)
    5. `/root/node_modules/moduleB/index.ts`
    6. `/root/node_modules/moduleB/index.tsx`
    7. `/root/node_modules/moduleB/index.d.ts`
    
    ---
    
    1. `/node_modules/moduleB.ts`
    2. `/node_modules/moduleB.tsx`
    3. `/node_modules/moduleB.d.ts`
    4. `/node_modules/moduleB/package.json` (如果指定了`"types"`属性)
    5. `/node_modules/moduleB/index.ts`
    6. `/node_modules/moduleB/index.tsx`
    7. `/node_modules/moduleB/index.d.ts`
    
    | 💡 TypeScript在步骤（8）和（15）会向上跳一级目录。
    
