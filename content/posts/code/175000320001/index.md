---
title: "npm中windows-build-tools下载问题"
date: "2025-06-16"
tags: ['NPM']
categories: ['动手实践']
---

在学习一些老旧的项目时下载各种的依赖但是下载的时候一直提示失败，显示python的依赖一直下载不下来。

通过以下命令就能成功下载

`npm --python_mirror=https://registry.npmmirror.com/-/binary/python/ install --global windows-build-tools`

```bash
Downloading python-2.7.15.amd64.msi
Error: getaddrinfo EAI_AGAIN www.python.org:443
Downloading Python failed. Error: { Error: getaddrinfo EAI_AGAIN www.python.org:443
    at Object._errnoException (util.js:992:11)
    at errnoException (dns.js:55:15)
    at GetAddrInfoReqWrap.onlookup [as oncomplete] (dns.js:92:26)
  code: 'EAI_AGAIN',
  errno: 'EAI_AGAIN',
  syscall: 'getaddrinfo',
  hostname: 'www.python.org',
  host: 'www.python.org',
  port: 443 }
windows-build-tools will now exit.
npm ERR! code ELIFECYCLE
npm ERR! errno 1
npm ERR! windows-build-tools@5.1.0 postinstall: `node ./dist/index.js`
npm ERR! Exit status 1
npm ERR!
npm ERR! Failed at the windows-build-tools@5.1.0 postinstall script.
npm ERR! This is probably not a problem with npm. There is likely additional logging output above.
```