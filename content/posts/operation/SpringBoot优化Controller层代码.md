---
title: "SpringBoot优化Controller层代码"
date: "2022-11-26"
tags: ['SpringBoot']
categories: ['动手实践']
---

本篇主要介绍的就是 `controller` 层的一些优雅技巧，一个完整的后端请求由4部分组成：1. `接口地址`(也就是URL地址)、2. `请求方式`(一般就是get、set，当然还有put、delete)、3. `请求数据`(request，有head跟body)、4. `响应数据`(response)

本篇将解决以下3个问题：
1. 当接收到请求时，如何优雅的校验参数
2. 返回响应数据该如何统一的进行处理
3. 接收到请求，处理业务逻辑时抛出了异常又该如何处理

## 统一返回格式

### 封装ResultVo

1. 首先先定义一个`状态码`的接口，所有`状态码`都需要实现它，有了标准才好做事
```java
public interface StatusCode {
    public int getCode();
    public String getMsg();
}
```

2. 与前端规定好自定义状态码
```java
@Getter
@AllArgsConstructor
@NoArgsConstructor
public enum ResultCode implements StatusCode {
    SUCCESS(1000, "请求成功"),
    FAILED(1001, "请求失败"),
    VALIDATE_ERROR(1002, "参数校验失败"),
    RESPONSE_PACK_ERROR(1003, "response返回包装失败");

    private int code;
    private String msg;
}
```

3. 定义 `ResultVo` 包装类
```java
@Data
public class ResultVo {
    // 状态码
    private int code;

    // 状态信息
    private String msg;

    // 返回对象
    private Object data;

    // 手动设置返回vo
    public ResultVo(int code, String msg, Object data) {
        this.code = code;
        this.msg = msg;
        this.data = data;
    }

    // 默认返回成功状态码，数据对象
    public ResultVo(Object data) {
        this.code = ResultCode.SUCCESS.getCode();
        this.msg = ResultCode.SUCCESS.getMsg();
        this.data = data;
    }

    // 返回指定状态码，数据对象
    public ResultVo(StatusCode statusCode, Object data) {
        this.code = statusCode.getCode();
        this.msg = statusCode.getMsg();
        this.data = data;
    }

    // 只返回状态码
    public ResultVo(StatusCode statusCode) {
        this.code = statusCode.getCode();
        this.msg = statusCode.getMsg();
        this.data = null;
    }
}
```

## 统一校验

### 引入依赖
由于新版 `Spring Boot` 已不内置校验模块，需自己引入相应的包
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

### 通过`@Validated`实现参数校验

```java
@Data
public class AddUserVo {

    @NotNull(message = "用户名不能为空")
    private String userName;

    @Length(min = 8, message = "密码最少需要8位")
    private String password;

    private Integer age;
}
```

```java
@PostMapping("/addUser")
public AddUserVo addUser(@Validated AddUserVo userVo) {
    return userVo;
}
```

虽然成功校验了参数，也返回了异常，但是返回的异常不符合预期，与我们之前定义了的统一状态码不一致，所以我们要进行优化一下，每次出现异常的时候，自动把状态码写好。

### 优化异常处理

我们可以通过控制台看出，校验参数抛出了什么异常
```java
Resolved [org.springframework.validation.BindException: org.springframework.validation.BeanPropertyBindingResult: 1 errors
```

我们看到代码抛出了`org.springframework.validation.BindException`的绑定异常，因此我们的思路就是AOP拦截所有controller，然后异常的时候统一拦截起来，进行封装！

但是我们可以通过 `Spring Boot` 提供的`@RestControllerAdvice`来增强所有`@RestController`，然后使用`@ExceptionHandler`注解，就可以拦截到对应的异常。这样开发起来就方便很多。

这里我们就拦截BindException.class就好了。最后在返回之前，我们对异常信息进行包装一下，包装成ResultVo，当然要跟上ResultCode.VALIDATE_ERROR的异常状态码。

```java
@RestControllerAdvice
public class ControllerExceptionAdvice {

    @ExceptionHandler({BindException.class})
    public ResultVo MethodArgumentNotValidExceptionHandler(BindException e){
        // 从异常对象中拿到ObjectError对象
        ObjectError objectError = e.getBindingResult().getAllErrors().get(0);
        return new ResultVo(ResultCode.VALIDATE_ERROR, objectError.getDefaultMessage());
    }
}
```

## 统一响应

### 统一包装响应

前面写的`ResultVo`导致每一个接口都需要写`new ResultVo(data)`，这样开发小哥肯定不乐意了，所以我们要优化封装，让`AOP`拦截所有`Controller`，然后再自动包一层`ResultVo`。

但是 `Spring Boot` 中也有现成的供我们使用：
```java
@RestControllerAdvice
public class ControllerResponseAdvice implements ResponseBodyAdvice<Object> {
    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        // response是ResultVo类型，或者注释了NotControllerResponseAdvice都不进行包装
        return !(returnType.getParameterType().isAssignableFrom(ResultVo.class));
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        // String类型不能直接包装
        if (returnType.getGenericParameterType().equals(String.class)){
            ObjectMapper objectMapper = new ObjectMapper();
            try {
                // 将数据包装在ResultVo里后转换为json串进行返回
                return objectMapper.writeValueAsString(new ResultVo(body));
            } catch (JsonProcessingException e) {
                throw new RuntimeException(e);
            }
        }
        // 否则直接包装成ResultVo返回
        return new ResultVo(body);
    }
}
```

1. `@RestControllerAdvice(basePackages = {"com.example"})`自动扫描了所有指定包下的`controller`，在`Response`时进行统一处理
2. 重写`supports`方法，也就是说，当返回类型已经是`ResultVo`了，那就不需要封装了，当不等与`ResultVo`时才进行调用`beforeBodyWrite`方法，跟过滤器的效果是一样的
3. 最后重写我们的封装方法`beforeBodyWrite`，注意除了`String`的返回值有点特殊，无法直接封装成json，我们需要进行特殊处理，其他的直接`new ResultVo(data);`就ok了

### NOT统一响应

有的时候，我们的系统不需要包装一层统一响应，比如项目中集成了一个`健康检测`的功能，这个时候是不需要包装统一响应的，导致最后接口对应不上。
```java
@GetMapping("/health")
public String health(Integer id) {
    return "success";
}
```

因为百分之99的请求还是需要包装的，只有个别不需要，写在包装的过滤器吧？又不是很好维护，那就加个注解好了。所有不需要包装的就加上这个注解。
```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface NotControllerResponseAdvice {
}
```

然后在我们的增强过滤方法上过滤包含这个注解的方法

```java
@RestControllerAdvice
public class ControllerResponseAdvice implements ResponseBodyAdvice<Object> {
    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        // response是ResultVo类型，或者注释了NotControllerResponseAdvice都不进行包装
        return !(returnType.getParameterType().isAssignableFrom(ResultVo.class) || returnType.hasMethodAnnotation(NotControllerResponseAdvice.class));
    }
    ...
```

最后就在不需要包装的方法上加上注解

```java
@GetMapping("/health")
@NotControllerResponseAdvice
public String health(Integer id) {
    return "success";
}
```

## 统一异常

每个系统都会有自己的`业务异常`，比如`库存不能小于0`子类的，这种异常并非程序异常，而是业务操作引发的异常，我们也需要进行规范的编排业务`异常状态码`，并且写一个专门处理的`异常类`，最后通过刚刚学习过的`异常拦截`统一进行处理，以及打`日志`。

1. 异常状态码枚举，既然是状态码，那就肯定要实现我们的标准接口`StatusCode`

```java
@Getter
@NoArgsConstructor
@AllArgsConstructor
public enum AppCode implements StatusCode {
    APP_ERROR(2000, "业务异常");

    private int code;
    private String msg;
}
```

2. 异常类

```java
@Getter
public class APIException extends RuntimeException {
    private int code;
    private String msg;

    // 手动设置异常
    public APIException(StatusCode statusCode, String message) {
        // message用于用户设置抛出错误详情，例如：当前价格-5，小于0
        super(message);
        // 状态码
        this.code = statusCode.getCode();
        // 状态码配套的msg
        this.msg = statusCode.getMsg();
    }

    // 默认异常使用APP_ERROR状态码
    public APIException(String message) {
        super(message);
        this.code = AppCode.APP_ERROR.getCode();
        this.msg = AppCode.APP_ERROR.getMsg();
    }
}
```

3. 最后进行统一异常的拦截

```java
@RestControllerAdvice
public class ControllerExceptionAdvice {

    @ExceptionHandler({BindException.class})
    public ResultVo MethodArgumentNotValidExceptionHandler(BindException e){
        // 从异常对象中拿到ObjectError对象
        ObjectError objectError = e.getBindingResult().getAllErrors().get(0);
        return new ResultVo(ResultCode.VALIDATE_ERROR, objectError.getDefaultMessage());
    }

    @ExceptionHandler(APIException.class)
    public ResultVo APIExceptionHandler(APIException e) {
        // log.error(e.getMessage(), e); 由于还没集成日志框架，暂且放着，写上TODO
        return new ResultVo(e.getCode(), e.getMsg(), e.getMessage());
    }
}
```

4. 最后使用，我们的代码只需要这么写

```java
@GetMapping("/error")
public String error(Integer id) {
    if (id.equals(0)){
        throw new APIException("用户不存在");
    }
    return "error";
}
```

就会自动抛出`AppCode.APP_ERROR`状态码的响应，并且带上异常详细信息`用户不存在`。