---
title: "JavaScript 代码片段"
date: "2023-06-26"
tags: ["JavaScript"]
categories: ["代码片段"]
---

## 从数组中删除重复项

```javascript
const numbers = [1, 1, 20, 3, 3, 3, 9, 9];
const uniqueNumbers = [...new Set(numbers)]; // -> [1, 20, 3, 9]
```

在 JavaScript 中，Set 是一个集合，它允许你仅存储唯一值。这意味着删除任何重复的值

展开运算符...将任何可迭代对象转换为数组

## 较短的 If-Else 的空合并

```javascript
let maybeSomething;


// LONG FORM
if(maybeSomething){
  console.log(maybeSomething)
} else {
  console.log("Nothing found")
}


// SHORTHAND
console.log(maybeSomething ?? "Nothing found")
```

nullish合并操作 ??，如果没有定义左侧返回右侧。如果是，则返回左侧

## 防止崩溃的可选链

```javascript
const student = {
  name: "Matt",
  age: 27,
  address: {
    state: "New York"
  },
};


// LONG FORM
console.log(student && student.address && student.address.ZIPCode); // Doesn't exist - Returns undefined


// SHORTHAND
console.log(student?.address?.ZIPCode); // Doesn't exist - Returns undefined
```

在未定义属性时使用可选链运算符，undefined将返回而不是错误。这可以防止你的代码崩溃

## 在没有第三个变量的情况下交换两个变量

```javascript
let x = 1;
let y = 2;


// LONGER FORM
let temp = x;
x = y;
y = temp;


// SHORTHAND
[x, y] = [y, x];
```

在 JavaScript 中，你可以使用解构从数组中拆分值

## 将任何值转换为布尔值

```javascript
!!true    // true
!!2       // true
!![]      // true
!!"Test"  // true


!!false   // false
!!0       // false
!!""      // false
```

在 JavaScript 中，你可以使用 !! 在 JS 中将任何内容转换为布尔值

## 扩展运算符

```javascript
const nums1 = [1, 2, 3];
const nums2 = [4, 5, 6];


// LONG FORM
let newArray = nums1.concat(nums2);


// SHORTHAND
newArray = [...nums1, ...nums2];
```

使用扩展运算符组合两个数组

```javascript
let numbers = [1, 2, 3];


// LONGER FORM
numbers.push(4);
numbers.push(5);


// SHORTHAND
numbers = [...numbers, 4, 5];
```

也可以使用此语法代替将值推送到数组

## 传播解构

```javascript
const student = {
  name: "Matt",
  age: 23,
  city: "Helsinki",
  state: "Finland",
};


// LONGER FORM
const name = student.name;
const age = student.age;
const address = { city: student.city, state: student.state };


// SHORTHAND
const { name, age, ...address } = student;
```

使用扩展运算符将剩余元素分配给变量

## 使用 && 进行短路评估

```javascript
var isReady = true;


function doSomething(){
  console.log("Yay!");
}


// LONGER FORM
if(isReady){
  doSomething();
}


// SHORTHAND
isReady && doSomething();
```

不必用if语句检查某事是否为真，你可以使用&&运算符

## 类固醇的字符串

```javascript
const age = 41;
const sentence = `I'm ${age} years old`;


// result: I'm 41 years old
```

通过将字符串包装在反引号内并${}用于嵌入值，从而在字符串之间插入变量

## 从数组中查找特定元素

```javascript
const fruits = [
  { type: "Banana", color: "Yellow" },
  { type: "Apple", color: "Green" }
];


// LONGER FORM
let yellowFruit;
for (let i = 0; i < fruits.length; ++i) {
  if (fruits[i].color === "Yellow") {
    yellowFruit = fruits[i];
  }
}


// SHORTHAND
yellowFruit = fruits.find((fruit) => fruit.color === "Yellow");
```

使用find()方法查找匹配特定条件的元素

## 对象属性赋值

```javascript
const name = "Luis", city = "Paris", age = 43, favoriteFood = "Spaghetti";

// LONGER FORM
const person = {
  name: name,
  city: city,
  age: age,
  favoriteFood: favoriteFood
};

// SHORTHAND
const person = { name, city, age, favoriteFood };
```

你是否希望对象键与值具有相同的名称？你可以省略对象文字来执行此操作

## 压缩 For 循环

```javascript
const numbers = [1, 2, 3, 4, 5];


// LONGER FORM
for(let i = 0; i < numbers.length; i++){
  console.log(numbers[i]);
}


// SHORTHAND
numbers.forEach(number => console.log(number));
```

使用内置forEach()方法通过一行代码循环遍历数组

## 默认功能参数

```javascript
// LONG FORM
function pickUp(fruit) {
  if(fruit === undefined){
    console.log("I picked up a Banana");
  } else {
    console.log(`I picked up a ${fruit}`);
  }
}

// SHORTHAND
function pickUp(fruit = "Banana") {
  console.log(`I picked up a ${fruit}`)
}

pickUp("Mango"); // -> I picked up a Mango
pickUp();        // -> I picked up a Banana
```

可以为函数参数提供默认值

## 将对象的值收集到数组中

```javascript
const info = { name: "Matt", country: "Finland", age: 35 };

// LONGER FORM
let data = [];
for (let key in info) {
  data.push(info[key]);
}

// SHORTHAND
const data = Object.values(info);
```

Object.values()将对象的所有值收集到一个新数组中

## 检查一个项目是否存在于数组中

```javascript
let numbers = [1, 2, 3];

// LONGER FORM
const hasNumber1 = numbers.indexOf(1) > -1 // -> True

// SHORTHAND/CLEANER APPROACH
const hasNumber1 = numbers.includes(1)     // -> True
```

可以使用 includes() 方法，而不是使用 indexOf() 方法来检查元素是否在数组中

## 压缩多个条件

```javascript
const num = 1;


// LONGER FORM
if(num == 1 || num == 2 || num == 3){
  console.log("Yay");
}


// SHORTHAND
if([1,2,3].includes(num)){
  console.log("Yay");
}
```

避免使用长|| 检查多个条件链，你可以使用你刚刚在上一个技巧中学到的东西——即，使用 includes() 方法

## 指数运算符

```javascript
// LONGER FORM
Math.pow(4,2); // 16
Math.pow(2,3); // 8


// SHORTHAND
4**2 // 16
2**3 // 8
```

## Math.floor() 简写

```javascript
// LONG FORM
Math.floor(5.25) // -> 5.0


// SHORTHAND
~~5.25 // -> 5.0
```

## 用一行代码分配多个值

```javascript
let num1, num2;


// LONGER FORM
num1 = 10;
num2 = 100;


// SHORTHAND
[num1, num2] = [10, 100];
```

```javascript
student = {
  name: "Matt",
  age: 29,
};


// LONGER FORM
let name = student.name;
let age = student.age;


// SHORTHAND
let { name, age } = student;
```

## 从url获取参数并转为对象

```javascript
const getParameters = URL => JSON.parse(`{"${decodeURI(URL.split("?")[1]).replace(/"/g, '\\"').replace(/&/g, '","').replace(/=/g, '":"')}"}`
  )

getParameters("https://www.google.com.hk/search?q=js+md&newwindow=1");
// {q: 'js+md', newwindow: '1'}
```

## 检查对象是否为空

```javascript
const isEmpty = obj => Reflect.ownKeys(obj).length === 0 && obj.constructor === Object;
isEmpty({}) // true
isEmpty({a:"not empty"}) //false
```

## 反转字符串

```javascript
const reverse = str => str.split('').reverse().join('');
reverse('this is reverse');
// esrever si siht
```

## 生成随机十六进制

```javascript
const randomHexColor = () => `#${Math.floor(Math.random() * 0xffffff).toString(16).padEnd(6, "0")}`
console.log(randomHexColor());
// #a2ce5b
```

## 检查当前选项卡是否在后台

```javascript
const isTabActive = () => !document.hidden;

isTabActive()
// true|false
```

## 检测元素是否处于焦点

```javascript
const elementIsInFocus = (el) => (el === document.activeElement);

elementIsInFocus(anyElement)
// 元素处于焦点返回true，反之返回false
```

## 检查设备类型

```javascript
const judgeDeviceType =
      () => /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|OperaMini/i.test(navigator.userAgent) ? 'Mobile' : 'PC';

judgeDeviceType()  // PC | Mobile
```

## 文字复制到剪贴板

```javascript
const copyText = async (text) => await navigator.clipboard.writeText(text)
copyText('单行代码 前端世界')
```

## 获取选定的文本

```javascript
const getSelectedText = () => window.getSelection().toString();

getSelectedText();
```

## 查询某天是否为工作日

```javascript
const isWeekday = (date) => date.getDay() % 6 !== 0;

isWeekday(new Date(2022, 03, 11))
```

## 转换华氏/摄氏

```javascript
// 华氏温度转换为摄氏温度
const fahrenheitToCelsius = (fahrenheit) => (fahrenheit - 32) * 5/9;

fahrenheitToCelsius(50);
// 10

// 摄氏温度转华氏温度
const celsiusToFahrenheit = (celsius) => celsius * 9/5 + 32;

celsiusToFahrenheit(100)
// 212
```

## 两日期之间相差的天数

```javascript
const dayDiff = (date1, date2) => Math.ceil(Math.abs(date1.getTime() - date2.getTime()) / 86400000);

dayDiff(new Date("2021-10-21"), new Date("2022-02-12"))
// Result: 114
```

## 将 RGB 转换为十六进制

```javascript
const rgbToHex = (r, g, b) =>   "#" + ((1 << 24) + (r << 16) + (g << 8) + b).toString(16).slice(1);

rgbToHex(255, 255, 255);
//  #ffffff
```

## 计算数组平均值

```javascript
const average = (arr) => arr.reduce((a, b) => a + b) / arr.length;

average([1,9,18,36]) //16
```
