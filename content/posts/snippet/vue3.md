---
title: "Vue3 代码片段"
date: "2024-05-16"
tags: ["Vue3"]
categories: ["代码片段"]
---

## Vue官方构建工具 `npm init vue@latest`

## 用ref()定义响应式变量

```javascript
const count = ref(0)

count.value++
```

## 计算属性

计算属性值会基于其响应式依赖被缓存、不要在计算函数中做异步请求或者更改 DOM、避免直接修改计算属性值

```javascript
const publishedBooksMessage = computed(() => {
  return author.books.length > 0 ? 'Yes' : 'No'
})
// {{ publishedBooksMessage }}
```

## Class 与 Style 绑定

Class绑定：`<MyComponent :class="{ active: isActive }" />`

Style绑定：`<div :style="{ color: activeColor, fontSize: fontSize + 'px' }"></div>`

## 侦听器

watch 的第一个参数可以是不同形式的“数据源”：它可以是一个 ref (包括计算属性)、一个响应式对象、一个 getter 函数、或多个数据源组成的数组：

```javascript
const x = ref(0)
const y = ref(0)

// 单个 ref
watch(x, (newX) => {
  console.log(`x is ${newX}`)
})

// getter 函数(当需要侦听响应式对象的属性时，需要使用getter函数：()=>obj.count)
watch(
  () => x.value + y.value,
  (sum) => {
    console.log(`sum of x + y is: ${sum}`)
  }
)

// 多个来源组成的数组
watch([x, () => y.value], ([newX, newY]) => {
  console.log(`x is ${newX} and y is ${newY}`)
})
```
`watchEffect()`：
watchEffect() 会立即执行一遍回调函数，如果这时函数产生了副作用，Vue 会自动追踪副作用的依赖关系，自动分析出响应源
```javascript
watchEffect(async () => {
  const response = await fetch(url.value)
  data.value = await response.json()
})
```

## Props

```javascript
const props = defineProps({
  // 基础类型检查
  // （给出 `null` 和 `undefined` 值则会跳过任何类型检查）
  propA: Number,
  // 多种可能的类型
  propB: [String, Number],
  // 必传，且为 String 类型
  propC: {
    type: String,
    required: true
  },
  // Number 类型的默认值
  propD: {
    type: Number,
    default: 100
  },
  // 对象类型的默认值
  propE: {
    type: Object,
    // 对象或数组的默认值
    // 必须从一个工厂函数返回。
    // 该函数接收组件所接收到的原始 prop 作为参数。
    default(rawProps) {
      return { message: 'hello' }
    }
  },
  // 自定义类型校验函数
  propF: {
    validator(value) {
      // The value must match one of these strings
      return ['success', 'warning', 'danger'].includes(value)
    }
  },
  // 函数类型的默认值
  propG: {
    type: Function,
    // 不像对象或数组的默认，这不是一个工厂函数。这会是一个用来作为默认值的函数
    default() {
      return 'Default function'
    }
  }
})

console.log(props.foo)
```

## 事件

```javascript
const emit = defineEmits(['inFocus', 'submit'])

function buttonClick() {
  emit('submit')
}
```

## 插槽

```html
// 使用组件
<FancyList :api-url="url" :per-page="10">
  <template #item="{ body, username, likes }">
    <div class="item">
      <p>{{ body }}</p>
      <p>by {{ username }} | {{ likes }} likes</p>
    </div>
  </template>
</FancyList>

// 组件定义
<ul>
  <li v-for="item in items">
    <slot name="item" v-bind="item"></slot>
  </li>
</ul>
```