---
title: "使用Nginx反向代理ONLYOFFICE"
date: "2025-06-24"
tags: ['Nginx', 'ONLYOFFICE']
categories: ['动手实践']
---

本文通过使用Nginx反向代理ONLYOFFICE来解决在Vue代码中写死网络路径，让修改ONLYOFFICE路径变得更加方便，不用再烦恼每次更新都需要修改代码。

## Vue代码

```Vue
<template>
  <DocumentEditor
    id="docEditor"
    documentServerUrl="http://documentserver/"
    :config="config"
  />
</template>
```

解决将`documentServerUrl`变成基于当前页面的反向代理，这样只需要修改Nginx配置就可以快速切换ONLYOFFICE服务端

```Vue
<template>
  <DocumentEditor
    id="docEditor"
    :documentServerUrl="window.location.origin + '/onlyoffice'"
    :config="config"
  />
</template>
```

## Nginx反向代理

> https://github.com/ONLYOFFICE/document-server-proxy/blob/master/nginx/proxy-to-virtual-path.conf

```conf
   location /onlyoffice/ {
        proxy_pass http://127.0.0.1:8881/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-Host $http_host/onlyoffice;
        proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
```

**划重点**

- `/onlyoffice/` `http://127.0.0.1:8881/` 最后的`/`一定需要
- `$http_host/onlyoffice` 中需要添加 `/onlyoffice`
- `$http_host` 比 `$host` 多添加了端口号！！！