---
title: "【vite插件】iconify字体图标离线加载"
date: "2025-06-25"
tags: ["Vite"]
categories: ["动手实践"]
---

本文介绍了如何在Vue项目中使用`vite-plugin-purge-icons`插件，实现离线加载`@iconify/json`中的字体图标，同时提供了一种预加载本地图标数据的方法以优化打包体积。

## 安装插件包

```bash
yarn add @iconify/iconify
yarn add @purge-icons/generated vite-plugin-purge-icons @iconify/json -D

# @iconify/json 中存放了离线字体

# vite-plugin-purge-icons 提供了离线扫描功能
# 扫描vue、ts、js等文件中插件正则匹配到的字体图标；
# 正常情况下，扫到的图标可以通过@iconify/json离线图标库加载
# 没扫到的图标会调用官方api远程加载
```

## 在编译阶段按需加载本地离线图标（注意：字体库会导致打包体积变大）

### 抽取要离线加载的json图标数据文件 `iconData.json` (在离线扫描失效时使用)

```json
{
  "ep:": ["add-location", "aim", "alarm-clock", "apple"],
  "fa:": ["home", "address-book", "address-book-o", "address-card"],
  "fa-solid:": ["abacus", "ad", "address-book"]
}
```

### `vite.config.js` 配置插件

```typescript
import PurgeIcons from 'vite-plugin-purge-icons'
import iconData form './data/iconData.json'

type ValueType<T> = [T, T[]]

// 提前加载本地图标数据
// 依赖 @iconify/json 包中提供的的数据
const preloadOfflineIcons = (iconData): string[] => {
  return Object.entries(iconData).reduce((a: string[], [k, v]: ValueType<string>) => {
    a = a.concat(v.map((i) =>  k+i))
    return a
  }, [])
}


// included中配置的图标不经过文件扫描，可通过@iconify/json处理后直接离线加载
export default {
  plugins: [
    PurgeIcons({
      included: preloadOfflineIcons()
    })
  ]
}
```

### `main.js` 加入离线打包支持

```typescript
import { createApp } from 'vue'
import App from './App.vue'

import '@purge-icons/generated' // <-- This

createApp(App).mount('#app')
```