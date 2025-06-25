---
title: "Vite打包时的错误：Cannot find module @rollup/rollup-linux-x64-gnu"
date: "2025-06-25"
tags: ["Vite"]
categories: ["动手实践"]
---

`vue3+vite`项目，在Windows环境开发时没有出现打包问题。

但是在Linux系统中打包报错，找不到依赖`rollup/rollup-linux-x64-gnu`。

报错信息如下：

```bash
Error: Cannot find module @rollup/rollup-linux-x64-gnu. npm has a bug related to optional dependencies (https://github.com/npm/cli/issues/4828). Please try `npm i` again after removing both package-lock.json and node_modules directory.
```

在`package.json`中加入以下配置项：

```json
  "optionalDependencies": {
    "@rollup/rollup-linux-x64-gnu": "*"
  },
```

问题解决