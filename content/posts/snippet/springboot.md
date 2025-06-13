---
title: "SpringBoot 代码片段"
date: "2023-06-16"
tags: ["SpringBoot"]
categories: ["代码片段"]
---

## Maven添加profiles

- profiles 标签

```xml
    <!-- 环境控制 -->
    <profiles>
        <!-- 开发 -->
        <profile>
            <id>dev</id>
            <activation>
                <!--默认激活配置-->
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
                <!--当前环境-->
                <profile.name>dev</profile.name>
            </properties>
        </profile>
        <!-- 生产 -->
        <profile>
            <id>prod</id>
            <properties>
                <!--当前环境-->
                <profile.name>prod</profile.name>
            </properties>
        </profile>
    </profiles>
```

- 打包添加resources

```xml
    <!-- 打包 -->
    <build>
        ......
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>
            </resource>
            <resource>
                <directory>src/main/java</directory>
                <includes>
                    <include>**/*.xml</include>
                    <include>**/*.json</include>
                    <include>**/*.ftl</include>
                </includes>
            </resource>
        </resources>
    </build>
```

- 创建对应文件

分别创建 `application.yml` 、 `application-dev.yml` 、 `application-prod.yml` 文件，dev、prod中写入需要需要区分环境的配置

- application.yml选中对应环境配置文件

```yml
spring:
  profiles:
    active: '@profile.name@'
```
