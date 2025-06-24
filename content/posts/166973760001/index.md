---
title: "Java中链式编程学习"
date: "2022-11-30"
tags: ['Java']
categories: ['编程知识']
---

## 简介

链式编程可以使得代码可读性高，链式编程的原理就是通过`return this`返回一个this对象，就是返回本身，达到链式效果。

## 实现链式编程

### 通过`return this`

```java
public class Book {
    private String bookId;
    private String title;
    private String cover;

    public String getBookId() {
        return bookId;
    }

    public Book setBookId(String bookId) {
        this.bookId = bookId;
        // 返回当前对象
        return this;
    }

    public String getTitle() {
        return title;
    }

    public Book setTitle(String title) {
        this.title = title;
        // 返回当前对象
        return this;
    }

    public String getCover() {
        return cover;
    }

    public Book setCover(String cover) {
        this.cover = cover;
        // 返回当前对象
        return this;
    }
}
```

代码比较简单，就是一个图书的模型，将三个字段的Set方法都返回了对象本身。

链式调用：
```java
Book book = new Book().setBookId("b.001").setTitle("庆余年").setCover("http://localhost/qyn.jpg");
```

### 静态初始化方法`of`

目前Spring中有很多方法提供了静态`of`方法，可以更方便的创建对象，结合`return this`将更为实用的实现链式编程，算是一种比较不错得扩展措施。在这种模式中，大量的使用了`of`替代了构造函数。

```java
public class Pageable {
    // 页码，从1开始
    private long page;

    // 页大小
    private int size;

    // 结果命中数量
    private long count;

    private Pageable(long page, int size) {
        this.page = page;
        this.size = size;
    }

    private Pageable(long page, int size, long count) {
        this(page, size);
        this.count = count;
    }

    // PageRequest时使用
    public static Pageable of(long page, int size) {
        return of(page, size, 0);
    }

    // 返回结果时，具有命中数量及返回总页码计算使用
    public static Pageable of(long page, int size, long count) {
        return new Pageable(page, size, count);
    }

    // 分页时，获取偏移量
    public long getOffset() {
        return (page - 1) * size;
    }

    // 返回总页码
    public long getPageCount() {
        return (long) Math.ceil((double) count / size);
    }

    public long getPage() {
        return this.page;
    }

    public int getSize() {
        return this.size;
    }

    public long getCount() {
        return this.count;
    }
}
```

上面示例中的分页工具类，只实现了三个成员变量`页码、页大小、条目数`，并且未实现成员变量的Set方法；构造方法全部为私有，实现了静态的`of`方法来创建分页对象。在该对象中同时提供了获取分页偏移量的`getOffset`方法，以及计算出总页码的`getPageCount`方法。

该工具一般有两种使用场景:

- 数据库查询后，返回数据列表以及分页信息，此时需要页码、页大小、条目数三个信息，使用及实现都比较方便
- 内存分页时，可以提供一个分页偏移量的计算方法

```java
Pageable pageable = Pageable.of(10, 10, 123);
System.out.println(pageable.getOffset());
System.out.println(pageable.getPageCount());
```

> 如果采用FastJson进行序列化，上述设计方式也能够满足返回JSON的需求，FastJson按照Get方法进行生成属性列表。

```java
System.out.println(JSON.toJSONString(Pageable.of(10, 10, 123)));
```

### Builder模式

Builder模式也是目前较为常用的一种链式编程方法，目前Spring中有大量的依据此模式进行的编码。以最为常见的`ResponseEntity`(或者Swagger模块也大量采用了此类方法)类来看，其内部定义了`Builder`接口，并默认实现了一个`DefaultBuild`内部类，内部方法采用`return this`实现链式模式，同时在`ResponseEntity`中提供了类似的`Builder`方法，如`ok、status`等静态方法。

从这些类的实现原理看，实现Builder模式也比较简单，一方面创建一个内部`Builder`类，实现相应信息的创建；同时在目标类中，实现一个静态的`builder`方法，用于创建`Builder`对象，并启动链式模式；最后再由内部类提供一个终止方法，该终止方法将完成目标类的创建。

```java
public class ApiInfo {
    private String name;
    private String description;
    private String url;
    private Map<String, String> params;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    public String getUrl() {
        return url;
    }

    public void setUrl(String url) {
        this.url = url;
    }

    public Map<String, String> getParams() {
        return params;
    }

    public void setParams(Map<String, String> params) {
        this.params = params;
    }

    @Override
    public String toString() {
        return "ApiInfo{" +
                "name='" + name + '\'' +
                ", description='" + description + '\'' +
                ", url='" + url + '\'' +
                ", params=" + params +
                '}';
    }

    public static ApiBuilder builder() {
        return new ApiBuilder();
    }

    // 内部Builder类
    public static class ApiBuilder {
        private String name;
        private String description;
        private String url;
        private Map<String, String> params;
        // name信息设置，链式
        public ApiBuilder name(String name) {
            this.name = name;
            return this;
        }
        // 注释信息设置
        public ApiBuilder description(String description) {
            this.description = description;
            return this;
        }

        public ApiBuilder url(String url) {
            this.url = url;
            return this;
        }
        // 参数信息设置
        public ApiBuilder params(String name, String description) {
            this.params = Optional.ofNullable(params).orElseGet(() -> Maps.newHashMap());
            this.params.put(name, description);

            return this;
        }
        // 创建ApiInfo对象
        public ApiInfo build() {
            ApiInfo apiInfo = new ApiInfo();
            apiInfo.setName(this.name);
            apiInfo.setDescription(this.description);
            apiInfo.setUrl(this.url);
            apiInfo.setParams(this.params);

            return apiInfo;
        }
    }
}
```

上述示例采用Builder模式实现了一个Api接口信息的对象设计，主要是能够设置接口名、地址以及相应的参数。内部设计了一个`ApiBuilder`，可以实现链式属性设置，并最终提供一个`build`方法用于终止时创建目标实例。

```java
ApiInfo apiInfo = ApiInfo.builder().name("获取资源信息")
        .description("根据资源ID获取资源信息")
        .url("/resource/:id")
        .params("id", "资源唯一ID")
        .params("token", "令牌")
        .build();
```

## lombok简化链式编程

链式编程比较简单，一般也就是`return`、`静态of`、`Builder`几种模式，可以直接编码实现，Spring以及一些开源项目Swagger等等都是这么做的。如果我们不是为了做一个通用的开源产品，只是业务性的编码，此时不需要通过大量的编码去实现链式编程，可以采用lombok进行实现，使用也比较简单。

> 使用lombok时，可以通过maven引入，也可以通过idea安装插件的方式引入。

### return 链式

仍然使用此前的例子`Book`类，进行改造，采用lombok注解进行编码。

> lombok实现`return this`链式，只需要在类添加注释`@Accessors(chain = true)`即可实现

```java
@Data
@Accessors(chain = true)
public class Book {
    private String bookId;
    private String title;
    private String cover;

    public static void main(String[] args) {
        Book book = new Book().setBookId("b.001").setTitle("庆余年").setCover("http://localhost/qyn.jpg");
        System.out.println(book);
    }
}
```

如果在IDE中，查看类接口(ctrl+o)，可以发现属性的Set方法，返回类型为`Book`，和自己编码的方案一致。
```java
Book book = new Book().setBookId("b.001").setTitle("庆余年").setCover("http://localhost/qyn.jpg");
```

### 静态初始化方法`of`

对Pageable类进行改造，使用lombok实现of方法。

静态`of`方法的添加，需要通过lombok的构造函数注解进行添加，`RequiredArgsConstructor`可以实现必备参数列表的构造，并通过`staticname`指定静态方法为`of`；针对上例中的Pageable对象提供了两种静态构造方法，一种是只需要指定页码和页大小即可，还有一种是全部参数的构造函数，可以计算总页码；`RequiredArgsConstructor`可以实现必备参数的构造；`AllArgsConstructor`实现全部参数的构造函数，并指定`staticName`为`of`，通过lombok的注解组合，实现Pageable与上例自己编码同等效果。

```java
@Data
@Accessors(chain = true)
@RequiredArgsConstructor(staticName = "of")
@AllArgsConstructor(staticName = "of")
public class Pageable {
    /**
     * 页码，从1开始
     */
    @NonNull
    private long page;

    /**
     * 页大小
     */
    @NonNull
    private int size;

    /**
     * 结果命中数量
     */
    private long count;

    /**
     * 分页时，获取偏移量
     */
    public long getOffset() {
        return (page - 1) * size;
    }

    /**
     * 返回总页码
     */
    public long getPageCount() {
        return (long) Math.ceil((double) count / size);
    }
}
```

通过组合注释实现了同样的效果，使用也和原有编码效果等同。

```java
Pageable pageable = Pageable.of(10, 10);
System.out.println(pageable.getOffset());
// 全部构造
System.out.println(JSON.toJSONString(Pageable.of(10, 10, 123)));
```

### Builder模式

对ApiInfo进行改造，使用lombok实现builder模式。

> lombok实现Builder模式，只要添加`@Builder`注解即可

```java
@Data
@Builder
public class ApiInfo {
    private String name;
    private String description;
    private String url;
    private Map<String, String> params;
}
```

使用上和之前的例子差不多，不过对于params默认只能是Set的形式。

```java
ApiInfo apiInfo = ApiInfo.builder().name("获取资源信息")
        .description("根据资源ID获取资源信息")
        .url("/resource/:id")
        .params(new HashMap<String, String>() {
            {
                put("id", "资源唯一ID");
                put("token", "令牌");
            }
        })
        .build();
```