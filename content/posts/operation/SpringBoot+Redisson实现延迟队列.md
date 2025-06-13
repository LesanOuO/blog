---
title: "SpringBoot+Redisson实现延迟队列"
date: "2023-07-01"
tags: ['SpringBoot', 'Redisson']
categories: ['动手实践']
---

在电商、支付等系统中，一般都是先创建订单（支付单），再给用户一定的时间进行支付，如果没有按时支付的话，就需要把之前的订单（支付单）取消掉。

这种类似的场景有很多，还有比如到期自动收货、超时自动退款、下单后自动发送短信等等都是类似的业务问题。

Redisson是一个在Redis的基础上实现的框架，它不仅提供了一系列的分布式的Java常用对象，还提供了许多分布式服务。Redission中定义了分布式延迟队列RDelayedQueue，这是一种基于我们前面介绍过的zset结构实现的延时队列，它允许以指定的延迟时长将元素放到目标队列中。

本文通过使用Redisson中的RDelayedQueue来实现一个简单的过期自动取消功能：

## 1. 自定义对象
```java
@Data
@AllArgsConstructor
public class DelayJobDto implements Serializable {
    private String username;
    private String msg;
}

```

## 2. 创建Bean
```java
@Configuration
public class DelayJobConfig {
    @Autowired
    private RedissonClient redissonClient;

    @Bean("blockQueue")
    public RBlockingDeque<DelayJobDto> getBlockQueue() {
        RBlockingDeque<DelayJobDto> blockingDeque = redissonClient.getBlockingDeque("delay:job:result");
        return blockingDeque;
    }

    @Bean("delayedQueue")
    public RDelayedQueue<DelayJobDto> getDelayQueue() {
        RBlockingDeque<DelayJobDto> blockingDeque = getBlockQueue();
        RDelayedQueue<DelayJobDto> delayedQueue = redissonClient.getDelayedQueue(blockingDeque);
        return delayedQueue;
    }
}
```

## 3. 编写到期处理服务
```java
@Slf4j
@Service
public class DelayJobService<T> {
    @Async
    public void expire(DelayJobDto delayJobDto) {
        log.info("用户：{} 已到期：msg：{}", delayJobDto.getUsername(), delayJobDto.getMsg());
    }
}
```

## 4. 创建线程
方式一：
```java
@Slf4j
@Component
public class DelayJobThread {
    @Autowired
    private DelayJobService<DelayJobDto> synthesisResultConsumer;

    @Autowired
    private RBlockingDeque<DelayJobDto> blockingDeque;

    @PostConstruct  // 当依赖注入完成后用于执行初始化的方法，并且只会被执行一次
    public void listen() {
        new Thread(()->{
            while (true) {
                try {
                    log.info("延时队列的数量：{}", blockingDeque.size());
                    log.info("listen 本次监听时间：{}", DateUtil.formatDate(new Date()));
                    DelayJobDto dto = blockingDeque.take(); //到期时自动取出
                    log.info("listen 从队列中获取需要查询结果的延时任务信息：{}", JSON.toJSONString(dto));
                    synthesisResultConsumer.expire(dto);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
}
```

方式二：
```java
@Slf4j
@Component
public class DelayJobRunner implements CommandLineRunner { // 程序启动后执行的功能
    @Autowired
    private DelayJobService<DelayJobDto> synthesisResultConsumer;

    @Autowired
    private RBlockingDeque<DelayJobDto> blockingDeque;

	@Override
	public void run(String... args) {
		new Thread(()->{
            while (true) {
                try {
                    log.info("延时队列的数量：{}", blockingDeque.size());
                    log.info("listen 本次监听时间：{}", DateUtil.formatDate(new Date()));
                    DelayJobDto dto = blockingDeque.take(); //到期时自动取出
                    log.info("listen 从队列中获取需要查询结果的延时任务信息：{}", JSON.toJSONString(dto));
                    synthesisResultConsumer.expire(dto);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
	}
}
```

## 5. 创建延时任务
```java
@Autowired
private RDelayedQueue<DelayJobDto> delayedQueue;

delayedQueue.offer(new DelayJobDto("Lesan", "延时任务"), 1, TimeUnit.MINUTES);
```

> 本文参考自：
> https://www.jianshu.com/p/3b6ad5d21e59