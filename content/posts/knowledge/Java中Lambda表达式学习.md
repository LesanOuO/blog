---
title: "Java中Lambda表达式学习"
date: "2022-12-03"
tags: ['Java']
categories: ['编程知识']
---

## 前言

在日常开发中，经常需要用到 `Java 8` 的`Lambda`表达式，它允许把函数作为一个方法的参数，让我们的代码更优雅、更简洁。本文主要总结收集一些常用的`Lambda`表达式的使用。

## list转map

```java
List<UserInfo> userInfoList = new ArrayList<>();
userInfoList.add(new UserInfo(1L, "hello", 18));
userInfoList.add(new UserInfo(2L, "world", 27));
userInfoList.add(new UserInfo(2L, "!", 26));

/**
  *  list 转 map
  *  使用Collectors.toMap的时候，如果有可以重复会报错，所以需要加(k1, k2) -> k1
  *  (k1, k2) -> k1 表示，如果有重复的key,则保留第一个，舍弃第二个
  */
Map<Long, UserInfo> userInfoMap = userInfoList.stream().collect(Collectors.toMap(UserInfo::getUserId, userInfo -> userInfo, (k1, k2) -> k1));
userInfoMap.values().forEach(a->System.out.println(a.getUserName()));
```

类似的，还有`Collectors.toList()`、`Collectors.toSet()`，表示把对应的流转化为list或者Set。

## filter()过滤

```java
List<UserInfo> userInfoList = new ArrayList<>();
userInfoList.add(new UserInfo(1L, "hello", 18));
userInfoList.add(new UserInfo(2L, "world", 27));
userInfoList.add(new UserInfo(2L, "!", 26));

/**
  * filter 过滤，留下超过18岁的用户
  */
List<UserInfo> userInfoResultList = userInfoList.stream().filter(user -> user.getAge() > 18).collect(Collectors.toList());
userInfoResultList.forEach(a -> System.out.println(a.getUserName()));
```

## foreach遍历

```java
/**
  * forEach 遍历集合List列表
  */
List<String> userNameList = Arrays.asList("hello", "world", "!");
userNameList.forEach(System.out::println);

/**
  *  forEach 遍历集合Map
  */
HashMap<String, String> hashMap = new HashMap<>();
hashMap.put("key1", "hello");
hashMap.put("key2", "world");
hashMap.put("key3", "!");
hashMap.forEach((k, v) -> System.out.println(k + ":\t" + v));
```

## groupingBy分组

```java
Map<String, List<UserInfo>> result = originUserInfoList.stream()
.collect(Collectors.groupingBy(UserInfo::getCity));
```

## sorted+Comparator 排序

```java
/**
  *  sorted + Comparator.comparing 排序列表，
  */
userInfoList = userInfoList.stream().sorted(Comparator.comparing(UserInfo::getAge)).collect(Collectors.toList());
userInfoList.forEach(a -> System.out.println(a.toString()));


/**
  * 如果想降序排序，则可以使用加reversed()
  */
userInfoList = userInfoList.stream().sorted(Comparator.comparing(UserInfo::getAge).reversed()).collect(Collectors.toList());
userInfoList.forEach(a -> System.out.println(a.toString()));
```

## distinct去重

```java
List<String> list = Arrays.asList("A", "B", "F", "A", "C");
List<String> temp = list.stream().distinct().collect(Collectors.toList());
temp.forEach(System.out::println);
```

## findFirst 返回第一个

```java
List<String> list = Arrays.asList("A", "B", "F", "A", "C");
list.stream().findFirst().ifPresent(System.out::println);
```

## anyMatch是否至少匹配一个元素

```java
Stream<String> stream = Stream.of("A", "B", "C", "D");
boolean match = stream.anyMatch(s -> s.contains("C"));
System.out.println(match);
//输出
true
```

## allMatch 匹配所有元素

```java
Stream<String> stream = Stream.of("A", "B", "C", "D");
boolean match = stream.allMatch(s -> s.contains("C"));
System.out.println(match);
//输出
false
```

## map转换

```java
List<String> list = Arrays.asList("hello", "world");
//转化为大写
List<String> upperCaselist = list.stream().map(String::toUpperCase).collect(Collectors.toList());
upperCaselist.forEach(System.out::println);
```

## Reduce

```java
int sum = Stream.of(1, 2, 3, 4).reduce(0, (a, b) -> a + b);
System.out.println(sum);
```

## peek 打印个日志

`peek()`方法是一个中间`Stream`操作，有时候我们可以使用`peek`来打印日志。

```java
List<String> result = Stream.of("hello", "world", "!")
            .filter(a -> a.contains("hello"))
            .peek(a -> System.out.println("say:" + a)).collect(Collectors.toList());
System.out.println(result);
```

## Max，Min最大最小

```java
Optional<UserInfo> maxAgeUserInfoOpt = userInfoList.stream().max(Comparator.comparing(UserInfo::getAge));
maxAgeUserInfoOpt.ifPresent(userInfo -> System.out.println("max age user:" + userInfo));

Optional<UserInfo> minAgeUserInfoOpt = userInfoList.stream().min(Comparator.comparing(UserInfo::getAge));
minAgeUserInfoOpt.ifPresent(userInfo -> System.out.println("min age user:" + userInfo));

```

## count统计

```java
long count = userInfoList.stream().filter(user -> user.getAge() > 18).count();
System.out.println("大于18岁的用户:" + count);
```

## 常用函数式接口

其实lambda离不开函数式接口，我们来看下JDK8常用的几个函数式接口：

- `Function<T, R>`（转换型）: 接受一个输入参数，返回一个结果
- `Consumer<T>` （消费型）: 接收一个输入参数，并且无返回操作
- `Predicate<T>` （判断型）: 接收一个输入参数，并且返回布尔值结果
- `Supplier<T>` （供给型）: 无参数，返回结果

### `Function<T, R>`

`Function<T, R>`是一个功能转换型的接口，可以把将一种类型的数据转化为另外一种类型的数据
```java
Function<String, Integer> function = String::length;
Stream<String> stream = Stream.of("hello", "world", "!");
Stream<Integer> resultStream = stream.map(function);
resultStream.forEach(System.out::println);
```

### `Consumer<T>`

`Consumer<T>`是一个消费性接口，通过传入参数，并且无返回的操作
```java
Consumer<String> comsumer = System.out::println;
Stream<String> stream = Stream.of("hello", "world", "!");
stream.forEach(comsumer);
```

### `Predicate<T>`

`Predicate<T>`是一个判断型接口,并且返回布尔值结果
```java
//获取每个字符串的长度，并且返回
Predicate<Integer> predicate = a -> a > 18;
UserInfo userInfo = new UserInfo(2L, "hello", 27);
System.out.println(predicate.test(userInfo.getAge()));
```

### `Supplier<T>`

`Supplier<T>`是一个供给型接口,无参数，有返回结果
```java
Supplier<Integer> supplier = () -> Integer.valueOf("666");
System.out.println(supplier.get());
```