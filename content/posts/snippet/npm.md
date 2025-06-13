---
title: "NPM 代码片段"
date: "2023-07-16"
tags: ["NPM"]
categories: ["代码片段"]
weight:
---

## Cannot read properties of null (reading ‘pickAlgorithm‘)

```bash
npm cache clear --force
```

## 单次 npm 使用代理和镜像

```bash
npm --registry https://registry.npmmirror.com --proxy http://xxx:9999 i
```
