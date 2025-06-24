---
title: "Maven解决Resource下office文件编译后损坏"
date: "2025-06-23"
tags: ["Maven"]
categories: ["编程知识"]
---

当我们再Resource目录下存放了office文件（如word、ppt、excel等），一经编译后会导致文件损坏无法打开。

原因：编译后会对文件进行转码

解决方法：通过`maven-resources-plugin`排除相关文件就可以解决问题，在pom文件中添加以下代码。
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-resources-plugin</artifactId>
    <version>${maven-resources-plugin.version}</version>
    <configuration>
        <nonFilteredFileExtensions>
            <nonFilteredFileExtension>xdb</nonFilteredFileExtension>
            <nonFilteredFileExtension>csv</nonFilteredFileExtension>
            <nonFilteredFileExtension>xlsx</nonFilteredFileExtension>
            <nonFilteredFileExtension>pdf</nonFilteredFileExtension>
            <nonFilteredFileExtension>docx</nonFilteredFileExtension>
            <nonFilteredFileExtension>pptx</nonFilteredFileExtension>
        </nonFilteredFileExtensions>
    </configuration>
</plugin>
```