---
title: "Git 代码片段"
date: "2022-11-26"
tags: ["Git"]
categories: ["代码片段"]
weight:
---

## 修改.gitignore立即生效

```bash
git rm -r –cached .
git add .
git commit -m "update .gitignore"
```