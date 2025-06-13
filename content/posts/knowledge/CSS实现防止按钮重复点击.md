---
title: "CSS实现防止按钮重复点击"
date: "2023-06-30"
tags: ['CSS']
categories: ['编程知识']
---

函数节流（throttle）是 JS 中一个非常常见的优化手段，可以有效的避免函数过于频繁的执行。

举个例子：一个保存按钮，为了避免重复提交或者服务器考虑，往往需要对点击行为做一定的限制，比如只允许每300ms提交一次，这时候我想大部分同学都会到网上直接拷贝一段throttle函数，或者直接引用lodash工具库
```javascript
btn.addEventListener('click', _.throttle(save, 300))
```
其实除了 JS 方式， CSS 也可以非常轻易的实现这样一个功能，无需任何框架库

## CSS实现思路分析

CSS 实现和 JS 的思维不同，需要从另一个角度去看待这个问题。

比如这里的需要对点击事件进行限制，也就是禁用点击事件，想想有什么方式可以禁用事件，没错，就是pointer-events;

然后是时间的限制，每次点击后需要自动禁用300ms，时间过后重新恢复，那么，有什么特性和时间以及状态恢复有关呢？没错，就是animation;

除此之外，还需要有触发时机，这里是点击行为，所以必然和伪类:active有关联。

因此，综合分析，实现这样一个功能需要用到pointer-events、animation以及:active，那么如何将这些思路串联起来呢？

其实这种场景可以理解成是对 CSS 动画的控制，比如有一个动画控制按钮从禁用->可点击的变化，每次点击时让这个动画重新执行一遍，在执行的过程中，一直处于禁用状态，是不是就达到了“节流”的效果了？

## CSS 动画的精准控制

假设有一个按钮，绑定了一个点击事件
```html
<button onclick="console.log('保存')">保存</button>
```
这时的按钮连续点击就会不断地触发

下面定义一个关于pointer-events的动画，就叫做 throttle 吧
```css
@keyframes throttle {
  from {
    pointer-events: none;
  }
  to {
    pointer-events: all;
  }
}
```
很简单吧，就是从禁用到可点击的变化。

接下来，将这个动画绑定在按钮上，这里为了方便测试，将动画设置成了2s
```css
button{
  animation: throttle 2s step-end forwards;
}
```
注意，这里动画的缓动函数设置成了阶梯曲线，step-end，它可以很方便的控制pointer-events的变化时间点。

如下示意，pointer-events在0~2秒内的值都是none，一旦到达2秒，就立刻变成了all，由于是forwards，会一直保持all的状态

最后，在点击时重新执行一遍动画，只需要在按下时设置动画为none就行了

实现如下
```css
button:active{
  animation: none;
}
```

为了演示方便，我们暂时把颜色变化也加在动画里
```css
@keyframes throttle {
  from {
    color: red;
    pointer-events: none;
  }
  to {
    color: green;
    pointer-events: all;
  }
}
```

现在如果文字是red，表示是禁用态，只有是green，才表示可以被点击，非常清晰明了

### 完整代码如下，就这么几行，如果需要改限制时间，直接改动画时间就行了
```css
button{
  animation: throttle 2s step-end forwards;
}
button:active{
  animation: none;
}
@keyframes throttle {
  from {
    pointer-events: none;
  }
  to {
    pointer-events: all;
  }
}
```

## CSS 实现的其他思路

具体思路是这样的，通过:active去触发transition变化，然后通过监听transition回调去动态设置按钮的禁用状态，实现如下

定义一个无关紧要的过渡属性，比如opacity
```css
button{
  opacity: .99;
  transition: opacity 2s;
}
button:not(:disabled):active{
  opacity: 1;
  transition: 0s;
}
```

然后监听transition的起始回调

```javascript

// 过渡开始
document.addEventListener('transitionstart', function(ev){
  ev.target.disabled = true
})
// 过渡结束
document.addEventListener('transitionend', function(ev){
  ev.target.disabled = false
})
```

这样做的最大好处是，这部分禁用的逻辑是完全和业务逻辑是解耦的，可以在任意时候，任意场合下无缝接入，也不受框架和环境影响

## 总结

以上通过 CSS 的思路实现了类似“节流”的功能，相比 JS 实现而言，实现更精简、使用更简单，没有框架限制，下面一起总结一下实现要点：

1. 函数节流是一个非常常见的优化方式，可以有效避免函数过于频繁的执行
2. CSS 的实现思路和 JS 不同，重点在于在于找到和该场景相关联的属性
3. CSS 实现“节流”其实就是控制一个动画的精准控制，假设有一个动画控制按钮从禁用->可点击的变化，每次点击时让这个动画重新执行一遍，在执行的过程中，一直处于禁用状态，这样就达到了“节流”的效果
4. 还可以通过 transition 的回调函数动态设置按钮禁用态
5. 这种实现的好处在于禁用逻辑和业务逻辑是完全解耦的

不过，这种实现方式还是比较有局限的，仅限于点击行为，像很多时候，节流可能会用在滚动事件或者键盘事件上，像这些场景就用传统方式实现就行了。