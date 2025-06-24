---
title: "SpringBoot自建starter库"
date: "2023-06-30"
tags: ['SpringBoot']
categories: ['动手实践']
---

本文用最短的时间给大家分享手写 starter 的完整流程。

## 创建项目
首先我们用 IDEA 开发工具来初始化一个 Spring Boot 项目，注意 Java 版本不要选太高、Spring Boot 版本不要选 3.x

## 引入依赖
初始化项目后，我们要在项目依赖文件 pom.xml 中引入几个核心依赖：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-configuration-processor</artifactId>
  <optional>true</optional>
</dependency>
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
  <optional>true</optional>
</dependency>
```
其中，spring-boot-autoconfigure 用于自动加载配置，spring-boot-configuration-processor 用于自动生成配置文件的自动提示。

有这些依赖就足够了，我们尽量保证 starter 的精简，便于其他项目引用时的兼容性。

此外，还要把 pom.xml 中的下面这段代码删掉：
```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
    </plugin>
  </plugins>
</build>
```

## 编写starter
接下来，假设我们已经开发了一个 Client（实现了主要功能的类），我们要编写一个配置类，用于自动创建 Client 实例。

```java
@Configuration
@ConfigurationProperties(prefix = "yuapi.client")
@Data
@ComponentScan
public class YuApiClientConfig {

    /**
     * appId
     */
    private String appId;

    /**
     * 秘钥
     */
    private String appSecret;

    @Bean
    public YuApiClient yuApiClient() {
        return new YuApiClient(appId, appSecret, userId);
    }
}
```

上述代码中，比较关键的注解是：

- @Configuration：告诉 Spring Boot 这是一个配置类，可以在该类中创建 Bean
- @ConfigurationProperties：和配置文件（一般是 application.yml）进行绑定，将配置文件中对应的配置映射到对象的属性中。比如 application.yml 中 yuapi.client.appId 的值会自动注入到 YuApiClientConfig 实例的 appId 属性。不用再把值硬编码到类中了！

写完这个配置类后，还要把它进行注册，创建一个配置文件 `resources/META_INF/spring.factories` ，编写如下代码：
```
# spring boot starter
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.yupi.mystarter.YuApiClientConfig
```

这样，就相当于给了项目一个启动入口（类似 main）。

然后，我们的 stater 类库就编写完毕啦！执行 mvn install 命令，就可以把它打包为本地依赖，供其他项目使用。

比如：
```xml

<dependency>
  <groupId>com.yupi</groupId>
  <artifactId>my-starter</artifactId>
  <version>0.0.1</version>
</dependency>
```