---
title: "搭建个人网站:GitHub+Vercel"
date: "2020-10-14"
tags: ["Vercel", "个人网站"]
categories: ["实用分享"]
weight: 1
---

本文主要介绍如何通过GitHub+Vercel免费搭建一个自己的静态网页。

### GitHub Pages

GitHub Pages是GitHub提供给大家的快速部署静态网页的功能，但是由于国内访问比较慢，这里提供一个相对较快的解决办法，就是用GitHub+Vercel。

### 第一步：注册登录Vercel

我想大家应该都有GitHub账号吧，这里就不多说了。

点击[Vercel官网](https://vercel.com/),并使用GitHub登录。

> 注意⚠️：GitHub账号不要使用QQ邮箱作为主邮箱。如果已经用QQ邮箱注册了GitHub，可以到Setting -> Emails里修改自己的主邮箱。

### 第二步：导入仓库

![](images/1.png)

它会让你选择一种登录Vercel的方法，支持使用GitHub，GitLab和Bitbucket登录，这里我们选GitHub，后面就是无脑Next的步骤。

顺利的话稍等片刻就会弹出部署成功的页面，还有浮夸的撒花。

部署完成之后可以点击visit进入网页看看效果。

### 第三步：配置域名

1.点击view Domains进行绑定自己的域名或者点击 `Settings`👉`Domains`👉输入自己的域名

2.输入自己的域名，然后点Add，它会弹出来一些需要做的配置，接下来需要去我们的域名提供商那里根据Vercel给出的要求进行配置。需要修改的有：Name Servers以及域名解析

最后等待等这两个改动都生效之后就可以用我们自己的域名访问刚刚建立的网站啦~

> 以后想要修改网站的话，只需要将改动push到GitHub上，vercel会自动把改动同步过来，完全不用管，超省心。
>
> 在一级域名配置好之后，也可以直接在vercel中使用二级域名，无需进行额外设置。
