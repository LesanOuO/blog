---
title: "Java中Function接口学习"
date: "2022-12-02"
tags: ['Java']
categories: ['编程知识']
---

## 前言

- `Function` 函数式接口
    - `Supplier`供给型函数
    - `Consumer`消费型函数
    - `Runnable`无参无返回型函数
    - `Function`函数的表现形式为接收一个参数，并返回一个值。`Supplier`、`Consumer`和`Runnable`可以看作`Function`的一种特殊表现形式

- 优化代码
    - 处理抛出异常的if
    - 处理if分支操作
    - 如果存在值执行消费操作，否则执行基于空的操作

在开发过程中经常会使用`if...else...`进行判断抛出异常、分支处理等操作。这些`if...else...`充斥在代码中严重影响了代码代码的美观，这时我们可以利用`Java 8`的`Function`接口来消灭`if...else...`。

## `Function` 函数式接口

使用注解`@FunctionalInterface`标识，并且只包含一个抽象方法的接口是函数式接口。函数式接口主要分为`Supplier`供给型函数、`Consumer`消费型函数、`Runnable`无参无返回型函数和`Function`有参有返回型函数。

### `Supplier`供给型函数

`Supplier`的表现形式为不接受参数、只返回数据

### `Consumer`消费型函数

`Consumer`消费型函数和`Supplier`刚好相反。`Consumer`接收一个参数，没有返回值

### `Runnable`无参无返回型函数

`Runnable`的表现形式为即没有参数也没有返回值

### `Function`函数

`Function`函数的表现形式为接收一个参数，并返回一个值。`Supplier`、`Consumer`和`Runnable`可以看作`Function`的一种特殊表现形式

## 代码优化

### 处理抛出异常的if

1. 定义函数

定义一个抛出异常的形式的函数式接口, 这个接口只有参数没有返回值是个消费型接口
```java
/**
 * 抛异常接口
 **/
@FunctionalInterface
public interface ThrowExceptionFunction {

    /**
     * 抛出异常信息
     **/
    void throwMessage(String message);
}
```

2. 编写判断方法

创建工具类`VUtils`并创建一个`isTure`方法，方法的返回值为刚才定义的`函数式接口`-`ThrowExceptionFunction`。`ThrowExceptionFunction`的接口实现逻辑为当参数`b`为`true`时抛出异常
```java
/**
 *  如果参数为true抛出异常
 **/
public static ThrowExceptionFunction isTure(boolean b){
    return (errorMessage) -> {
        if (b){
            throw new RuntimeException(errorMessage);
        }
    };
}
```

3. 使用方式

调用工具类参数参数后，调用函数式接口的throwMessage方法传入异常信息
```java
// 当出入的参数为false时正常执行
VUtils.isTure(false).throwMessage("error");
// 当出入的参数为true时抛出异常
VUtils.isTure(true).throwMessage("error");
```

### 处理if分支操作

1. 定义函数式接口

创建一个名为`BranchHandle`的函数式接口，接口的参数为两个`Runnable`接口。这两个两个`Runnable`接口分别代表了为`true`或`false`时要进行的操作
```java
/**
 * 分支处理接口
 **/
@FunctionalInterface
public interface BranchHandle {

    /**
     * 分支操作
     **/
    void trueOrFalseHandle(Runnable trueHandle, Runnable falseHandle);

}
```

2. 编写判断方法

创建一个名为`isTureOrFalse`的方法，方法的返回值为刚才定义的`函数式接口`-`BranchHandle`
```java
/**
 * 参数为true或false时，分别进行不同的操作
 **/
public static BranchHandle isTureOrFalse(boolean b){
    return (trueHandle, falseHandle) -> {
        if (b){
            trueHandle.run();
        } else {
            falseHandle.run();
        }
    };
}
```

3. 使用方式

```java
VUtils.isTureOrFalse(true)
        .trueOrFalseHandle(()->{
            System.out.println("true"); // 参数为true时，执行trueHandle
        }, ()->{
            System.out.println("false"); // 参数为false时，执行falseHandle
        });
```

### 参数为false时，执行falseHandle

1. 定义函数

创建一个名为`PresentOrElseHandler`的函数式接口，接口的参数一个为`Consumer`接口。一个为`Runnable`,分别代表值不为空时执行消费操作和值为空时执行的其他操作
```java
/**
 * 空值与非空值分支处理
 */
public interface PresentOrElseHandler<T extends Object> {

    /**
     * 值不为空时执行消费操作
     * 值为空时执行其他的操作
     **/
   void presentOrElseHandle(Consumer<? super T> action, Runnable emptyAction);
}
```

2. 编写判断方法

创建一个名为`isBlankOrNoBlank`的方法，方法的返回值为刚才定义的`函数式接口`-`PresentOrElseHandler`
```java
/**
 * 参数为true或false时，分别进行不同的操作
 **/
public static PresentOrElseHandler<?> isBlankOrNoBlank(String str){
    return (consumer, runnable) -> {
        if (str == null || str.length() == 0){
            runnable.run();
        } else {
            consumer.accept(str);
        }
    };
}
```

3. 使用方式

调用工具类参数参数后，调用函数式接口的`presentOrElseHandle`方法传入一个`Consumer`和`Runnable`

```java
// 参数不为空时，打印参数
VUtils.isBlankOrNoBlank("hello")
        .presentOrElseHandle(System.out::println,()->{
            System.out.println("blank");
        });
// 参数不为空时
VUtils.isBlankOrNoBlank("")
.presentOrElseHandle(System.out::println,()->{
    System.out.println("blank");
});
```