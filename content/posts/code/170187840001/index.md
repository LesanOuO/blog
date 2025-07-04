---
title: "如何更优雅地实现策略模式"
date: "2023-12-07"
tags: ['设计模式']
categories: ['编程知识']
---

## 什么是策略模式

策略模式，应该是工作中比较常用的设计模式，调用方自己选择用哪一种策略完成对数据的操作，也就是“一个类的行为或其算法可以在运行时更改”，将一些除了过程不同其他都一样的函数封装成策略，然后调用方自己去选择想让数据执行什么过程策略。常见的例子为根据用户分类推荐不同的排行榜（用户关注点不一样，推荐榜单就不一样）

和单例模式一样，随着时间发展，不再推荐经典策略模式，更推荐简单策略用枚举策略模式，复杂地用工厂策略模式。下面引入一个例子，我们的需求是：对一份股票数据列表，给出低价榜、高价榜、涨幅榜。这其中只有排序条件的区别，比较适合作为策略模式的例子

## 经典策略模式

数据DTO

```java
@Data
public class Stock {

    // 股票交易代码
    private String code;

    // 现价
    private Double price;

    // 涨幅
    private Double rise;
}
```

抽象得到的策略接口

```java
public interface Strategy {

    /**
     * 将股票列表排序
     *
     * @param source 源数据
     * @return 排序后的榜单
     */
    List<Stock> sort(List<Stock> source);
}
```

实现我们的策略类

```java

/**
 * 高价榜
 */
public class HighPriceRank implements Strategy {

    @Override
    public List<Stock> sort(List<Stock> source) {
        return source.stream()
                .sorted(Comparator.comparing(Stock::getPrice).reversed())
                .collect(Collectors.toList());
    }
}

/**
 * 低价榜
 */
public class LowPriceRank implements Strategy {

    @Override
    public List<Stock> sort(List<Stock> source) {
        return source.stream()
                .sorted(Comparator.comparing(Stock::getPrice))
                .collect(Collectors.toList());
    }
}

/**
 * 高涨幅榜
 */
public class HighRiseRank implements Strategy {

    @Override
    public List<Stock> sort(List<Stock> source) {
        return source.stream()
                .sorted(Comparator.comparing(Stock::getRise).reversed())
                .collect(Collectors.toList());
    }
}
```

经典的Context类

```java
public class Context {
    private Strategy strategy;

    public void setStrategy(Strategy strategy) {
        this.strategy = strategy;
    }

    public List<Stock> getRank(List<Stock> source) {
        return strategy.sort(source);
    }
}
```

于是 我们顺礼成章地得到调用类--榜单实例 `RankServiceImpl`

```java
@Service
public class RankServiceImpl {

    /**
     * dataService.getSource() 提供原始的股票数据
     */
    @Resource
    private DataService dataService;

    /**
     * 前端传入榜单类型, 返回排序完的榜单
     *
     * @param rankType 榜单类型
     * @return 榜单数据
     */
    public List<Stock> getRank(String rankType) {
        // 创建上下文
        Context context = new Context();
        // 这里选择策略
        switch (rankType) {
            case "HighPrice":
                context.setStrategy(new HighPriceRank());
                break;
            case "LowPrice":
                context.setStrategy(new LowPriceRank());
                break;
            case "HighRise":
                context.setStrategy(new HighRiseRank());
                break;
            default:
                throw new IllegalArgumentException("rankType not found");
        }
        // 然后执行策略
        return context.getRank(dataService.getSource());
    }
}
```

我们可以看到经典方法，创建了一个接口、三个策略类，还是比较啰嗦的。调用类的实现也待商榷，新增一个策略类还要修改榜单实例（可以用抽象工厂解决，但是复杂度又上升了）。加之我们有更好的选择，所以此处不再推荐经典策略模式。

## 基于枚举的策略模式

这里对这种简单的策略，推荐用枚举进行优化。枚举的本质是创建了一些静态类的集合。

我下面直接给出例子，大家可以直观感受一下

枚举策略类

```java
public enum RankEnum {
    // 以下三个为策略实例
    HighPrice {
        @Override
        public List<Stock> sort(List<Stock> source) {
            return source.stream()
                    .sorted(Comparator.comparing(Stock::getPrice).reversed())
                    .collect(Collectors.toList());
        }
    },
    LowPrice {
        @Override
        public List<Stock> sort(List<Stock> source) {
            return source.stream()
                    .sorted(Comparator.comparing(Stock::getPrice))
                    .collect(Collectors.toList());
        }
    },
    HighRise {
        @Override
        public List<Stock> sort(List<Stock> source) {
            return source.stream()
                    .sorted(Comparator.comparing(Stock::getRise).reversed())
                    .collect(Collectors.toList());
        }
    };

    // 这里定义了策略接口
    public abstract List<Stock> sort(List<Stock> source);
}
```

对应的调用类也得以优化，榜单实例 `RankServiceImpl`

```java
@Service
public class RankServiceImpl {

    /**
     * dataService.getSource() 提供原始的股票数据
     */
    @Resource
    private DataService dataService;

    /**
     * 前端传入榜单类型, 返回排序完的榜单
     *
     * @param rankType 榜单类型 形似 RankEnum.HighPrice.name()
     * @return 榜单数据
     */
    public List<Stock> getRank(String rankType) {
        // 获取策略，这里如果未匹配会抛 IllegalArgumentException异常
        RankEnum rank = RankEnum.valueOf(rankType);
        // 然后执行策略
        return rank.sort(dataService.getSource());
    }
}
```

可以看到，如果策略简单的话，基于枚举的策略模式优雅许多，调用方也做到了0修改，但正确地使用枚举策略模式需要额外考虑以下几点。

- 枚举的策略类是公用且静态，这意味着这个策略过程不能引入非静态的部分，扩展性受限

- 策略模式的目标之一，是优秀的扩展性和可维护性，最好能新增或修改某一策略类时，对其他类是无改动的。而枚举策略如果过多或者过程复杂，维护是比较困难的，可维护性受限

## 基于工厂的策略模式

为了解决良好的扩展性和可维护性，我更推荐以下利用spring自带beanFactory的优势，实现一个基于工厂的策略模式。
策略类改动只是添加了`@Service`注解，并指定了Service的value属性

```java
/**
 * 高价榜
 * 注意申明 Service.value = HighPrice,他是我们的key,下同
 */
@Service("HighPrice")
public class HighPriceRank implements Strategy {

    @Override
    public List<Stock> sort(List<Stock> source) {
        return source.stream()
                .sorted(Comparator.comparing(Stock::getPrice).reversed())
                .collect(Collectors.toList());
    }
}

/**
 * 低价榜
 */
@Service("LowPrice")
public class LowPriceRank implements Strategy {

    @Override
    public List<Stock> sort(List<Stock> source) {
        return source.stream()
                .sorted(Comparator.comparing(Stock::getPrice))
                .collect(Collectors.toList());
    }
}

/**
 * 高涨幅榜
 */
@Service("HighRise")
public class HighRiseRank implements Strategy {

    @Override
    public List<Stock> sort(List<Stock> source) {
        return source.stream()
                .sorted(Comparator.comparing(Stock::getRise).reversed())
                .collect(Collectors.toList());
    }
}
```

调用类修改较大，接入借助spring工厂特性，完成策略类

```java
@Service
public class RankServiceImpl {

    /**
     * dataService.getSource() 提供原始的股票数据
     */
    @Resource
    private DataService dataService;
    /**
     * 利用注解@Resource和@Autowired特性,直接获取所有策略类
     * key = @Service的value
     */
    @Resource
    private Map<String, Strategy> rankMap;

    /**
     * 前端传入榜单类型, 返回排序完的榜单
     *
     * @param rankType 榜单类型 和Service注解的value属性一致
     * @return 榜单数据
     */
    public List<Stock> getRank(String rankType) {
        // 判断策略是否存在
        if (!rankMap.containsKey(rankType)) {
            throw new IllegalArgumentException("rankType not found");
        }
        // 获得策略实例
        Strategy rank = rankMap.get(rankType);
        // 执行策略
        return rank.sort(dataService.getSource());
    }
}
```

工厂策略模式会比枚举策略模式啰嗦，但也更加灵活、易扩展性和易维护。故简单策略推荐枚举策略模式，复杂策略才推荐工厂策略模式。