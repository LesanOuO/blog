---
title: "如何在.NET中开启静态文件访问"
date: "2022-07-17"
tags: ['.NET']
categories: ['动手实践']
---

## 基本使用

静态文件（常见的有HTML，CSS，图片和JS等资源）可以通过 .NET Core 应用直接提供给客户端。还有一些比较常用的文件（PDF或者一些需要下载的文件）也是需要通过静态文件的方式提供下载的，如果没有搭建文件管理的服务器的话，通过静态文件的方式下载也是不错的选择。

静态文件一般位于网站根目录的 `wwwroot` 文件夹下，可以通过相对根的路径来访问文件夹中的文件，如 `http://ip:port/filename`

使用静态文件服务前，需要配置中间件，把静态文件中间件加入到管道中。静态文件一般会默认配置，在Configure方法中调用`app.UseStaticFiles()`。`app.UseStaticFiles()` 使得web root(默认为wwwroot)下的文件可以被访问。同时可以通过UseStaticFiles方法将其他目录下的内容也可以向外提供：
```csharp
// 假如wwwroot外面有一个MyStaticFiles文件夹，要访问文件夹里面的资源
app.UseStaticFiles(new StaticFileOptions() {
    FileProvider = new PhysicalFileProvider(
        Path.Combine(Directory.GetCurrentDirectory(), @"MyStaticFiles")), //用于定位资源的文件系统
    RequestPath = new PathString("/StaticFiles") //请求地址
});
// 现在可以通过http://ip:port/StaticFiles/filename 来访问文件夹下的内容
```

## 静态文件授权

静态文件组件默认不提供授权检查。任何通过静态文件中间件访问的文件都是公开的。要想给文件授权，可以将文件保存在wwwroot之外，并将目录设置为可被静态文件中间件能够访问，同时通过一个controller action来访问文件，在action中授权后返回`FileResult`。

## 目录浏览

目录浏览允许网站用户看到指定目录下的目录和文件列表。基于安全考虑，默认情况下是禁止目录访问功能。在Startup中Configure方法调用`UseDirectoryBrowser`扩展方法可以开启网络应用目录浏览：

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    app.UseStaticFiles();

    app.UseDirectoryBrowser(new DirectoryBrowserOptions() {
        FileProvider = new PhysicalFileProvider(
            Path.Combine(Directory.GetCurrentDirectory(),@"wwwroot\images")),
        RequestPath = new PathString("/MyImages") //如果不指定RequestPath，会将PhysicalFileProvider中的路径参数作为默认文件夹，替换掉wwwroot
    });
}
```

然后在Startup中CongigureServices方法调用`AddDirectoryBrowser`扩展方法，这样就可以通过访问`http://<app>/MyImages浏览wwwroot/images`文件夹中的目录，但是不能访问文件：

![](https://cdn.jsdelivr.net/gh/LesanOuO/images/img/DotNetCore静态文件1.png)

要想访问具体文件需要调用UseStaticFiles配置：

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    app.UseStaticFiles();
    app.UseStaticFiles(new StaticFileOptions() {
        FileProvider = new PhysicalFileProvider(
            Path.Combine(Directory.GetCurrentDirectory(), @"wwwroot\images")), //用于定位资源的文件系统
        RequestPath = new PathString("/MyImages")
    });
    app.UseDirectoryBrowser(new DirectoryBrowserOptions() {
        FileProvider = new PhysicalFileProvider(
            Path.Combine(Directory.GetCurrentDirectory(),@"wwwroot\images")),
        RequestPath = new PathString("/MyImages")
    });
}
```
## UseFileServer

通过 `UseFileServer` 集合了`UseStaticFiles`,`UseDefaultFiles`,`UseDirectoryBrowser`。可以实现类似文件服务器的功能

调用`app.UseFileServer();` 可以获得静态文件和默认文件，但不允许直接访问目录。需要调用`app.UseFileServer(enableDirectoryBrowsing:true);` 才能启用目录浏览功能。

如果想要访问wwwroot以外的文件，需要配置一个FileServerOptions对象
```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    app.UseStaticFiles();//如果不调用，将不会启动默认功能。
    app.UseFileServer(new FileServerOptions()
    {
        FileProvider = new PhysicalFileProvider(
            Path.Combine(Directory.GetCurrentDirectory(), @"MyStaticFiles")),
        RequestPath = new PathString("/StaticFiles"),
        EnableDirectoryBrowsing = true
    });
}
```

注意，如果将enableDirectoryBrowsing设置为true，需要在ConfigureServices中调用`services.AddDirectoryBrowser()`;如果默认文件夹下有默认页面，将显示默认页面，而不是目录列表。

## FileExtensionContentTypeProvider

FileExtensionContentTypeProvider类包含一个将文件扩展名映射到MIME内容类型的集合。

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    var provider = new FileExtensionContentTypeProvider();
    provider.Mappings[".htm3"] = "text/html";
    provider.Mappings["images"] = "iamge/png";
    provider.Mappings.Remove(".mp4");

    app.UseStaticFiles(new StaticFileOptions() {
        FileProvider = new PhysicalFileProvider(
            Path.Combine(Directory.GetCurrentDirectory(), @"MyStaticFiles")),
        RequestPath = new PathString("/StaticFiles"),
        ContentTypeProvider = provider
    });
}
```

## 非标准的内容类型

如果用户请求了一个未知的文件类型，静态文件中间件将会返回HTTP 404响应。如果启用目录浏览，则该文件的链接将会被显示，但URL会返回一个HTTP404错误。

使用UseStaticFiles方法可以将未知类型作为指定类型处理：

```csharp
app.UseStaticFiles(new StaticFileOptions() {
    ServeUnknownFileTypes = true,
    DefaultContentType = "application/x-msdownload"
});
```
对于未识别的，默认为application/x-msdownload，浏览器将会下载这些文件。

> 本文参考自：https://www.jb51.net/article/244297.htm