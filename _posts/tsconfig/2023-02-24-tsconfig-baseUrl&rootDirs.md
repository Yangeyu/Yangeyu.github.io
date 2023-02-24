---
layout: post
title: "tsconfig.json - baseUrl & rootDirs"
subtitle: '介绍baseUrl和paths配置别名，rootDirs功能说明'
author: "Vascent"
header-img: "//images.unsplash.com/photo-1620503374956-c942862f0372?ixlib=rb-4.0.3&q=80&fm=jpg&crop=entropy&cs=tinysrgb&w=4800"
<!-- header-style: text -->
tags:
  - tsconfig.json
  - TypeScript
---

# tsconfig - baseUrl & rootDirs

### Overview

有时工程源码结构与输出结构不同。 通常是要经过一系统的构建步骤最后生成输出。 它们包括

- 将 `.ts`编译成`.js`，
- 将不同位置的依赖拷贝至一个输出位置。

最终结果就是

- 运行时的模块名与包含它们声明的源文件里的模块名不同。
- 或者**最终输出文件里**的模块路径与**编译时**的源文件路径不同了。

> ❗ 编译器*不会*进行这些转换操作； 它只是利用这些信息来指导模块的导入。

---

设置`baseUrl`来告诉编译器到哪里去查找模块。 所有**非相对模块导入**都会被当做相对于 `baseUrl`。

`baseUrl` **的值由以下两者之一决定：

- 命令行中*baseUrl*的值（如果给定的路径是相对的，那么将相对于当前路径进行计算）
- ‘tsconfig.json’里的*baseUrl*属性（如果给定的路径是相对的，那么将相对于‘tsconfig.json’路径进行计算）

> ❗ 注意**相对模块**的导入不会被设置的`baseUrl`所影响，因为它们总是相对于导入它们的文件。

### 路径映射

**Example1:**

工程结构可能如下：

```
projectRoot
├── folder1
│   ├── file1.ts (imports 'folder1/file2' and 'folder2/file3')
│   └── file2.ts
├── generated
│   ├── folder1
│   └── folder2
│       └── file3.ts
└── tsconfig.json
```

- ***解析流程***
  - 导入'folder1/file2'
    1. 匹配'*'模式且通配符捕获到整个名字。
    2. 尝试列表里的第一个替换：'*' -> `folder1/file2`
    3. 替换结果为非相对名 - 与*baseUrl*合并 -> `projectRoot/folder1/file2.ts`
    4. 文件存在。完成。
  - 导入'folder2/file3'
    1. 匹配'*'模式且通配符捕获到整个名字
    2. 尝试列表里的第一个替换：'*' -> `folder2/file3`
    3. 替换结果为非相对名 - 与*baseUrl*合并 -> `projectRoot/folder2/file3.ts`
    4. 文件不存在，跳到第二个替换。
    5. 第二个替换：'generated/*' -> `generated/folder2/file3`
    6. 替换结果为非相对名 - 与*baseUrl*合并 -> `projectRoot/generated/folder2/file3.ts`
    7. 文件存在。完成。

`tsconfig.json`文件如下：

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "*": [
        "*",
        "generated/*"
      ]
    }
  }
}
```

它告诉编译器所有匹配`"*"`（所有的值）模式的模块导入会在以下两个位置查找：

1. `"*"`： 表示名字不发生改变，所以映射为`<moduleName>` => `<baseUrl>/<moduleName>`
2. `"generated/*"`表示模块名添加了“generated”前缀，所以映射为`<moduleName>` => `<baseUrl>/generated/<moduleName>`

补充：

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
        "/@/*": ["src/*"],
        "/#/*": ["types/*"]
     }
  }
}
```

### 利用`rootDirs`指定虚拟目录

- 告诉编译器生成这个虚拟目录的*roots*
- 编译器可以在“虚拟”目录下解析相对模块导入，就 *好像* 它们被合并在了一起一样。

**Example:**

工程目录：

```
src
 └── views
     └── view1.ts (can import "./template1", "./view2`)
     └── view2.ts (can import "./template1", "./view1`)

 generated
 └── templates
         └── views
             └── template1.ts (can import "./view1", "./view2")
```

`tsconfig.json` 配置：

```json
{
  "compilerOptions": {
    "rootDirs": ["src/views", "generated/templates/views"]
  }
}
```

虚拟结果：

```
.
├── template1.ts (can import "./view1", "./view2")
├── view1.ts (can import "./template1", "./view2`)
└── view2.ts (can import "./template1", "./view1`)
```
