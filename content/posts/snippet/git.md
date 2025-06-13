---
title: "Git 代码片段"
date: "2023-09-26"
tags: ["Git"]
categories: ["代码片段"]
---

## 修改.gitignore立即生效

```bash
git rm -r –cached .
git add .
git commit -m "update .gitignore"
```