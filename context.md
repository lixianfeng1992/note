# 执行上下文（Execution context stack，ECS）
```js
executionContextObj = {
  scopeChain: "作用域链, 跟闭包相关",
  VO: "变量对象",
  this: "经典的 this"
}
```
每当解释器遇到一段代码都会向执行栈中压入相应的上下文

# 变量对象
- 变量对象是与执行上下文相关的数据作用域，存储了在上下文中定义的变量和函数声明
# 不同上下文变量对象的区别
## 全局上下文
```js
var a = 10;
console.log(this.a) // 10
console.log(window.a) // 10
```
+ 全局上下文中的变量对象就是全局对象

## 函数上下文
+ 在函数上下文中，我们用活动对象(activation object, AO)来表示变量对象。
+ 活动对象和变量对象其实是一个东西，只是变量对象是规范上的或者说是引擎实现上的，不可在 JavaScript 环境中访问，只有到当进入一个执行上下文中，这个执行上下文的变量对象才会被激活，所以才叫 activation object 呐，而只有被激活的变量对象，也就是活动对象上的各种属性才能被访问。
+ 活动对象是在进入函数上下文时刻被创建的，它通过函数的 arguments 属性初始化。arguments 属性值是 Arguments 对象


# 执行上下文的步骤
## 创建
1. 初始化作用域链
2. 创建变量对象
  + 创建arguments
  + 扫描函数声明
  + 扫描变量声明
3. 求this
## 执行阶段
1. 初始化变量和函数的引用
2. 执行代码


## 创建变量对象
1. 函数的所有形参 (如果是函数上下文)
+ 由名称和对应值组成的一个变量对象的属性被创建
+ 没有实参，属性值设为 undefined

2. 扫描函数声明
+ 由名称和对应值（函数对象(function-object)）组成一个变量对象的属性被创建
+ 如果变量对象已经存在相同名称的属性，则完全替换这个属性

3. 扫描变量声明
+ 由名称和对应值（undefined）组成一个变量对象的属性被创建；
+ 如果变量名称跟已经声明的形式参数或函数相同，则变量声明不会干扰已经存在的这类属性

```js
function foo(a) {
  var b = 2;
  function c() {};
  var d = function() {};
  b = 3;
}

foo(1)
```
```js
AO = {
  arguments: {
    0: 1,
    length: 1
  },
  a: 1,
  b: undefined,
  c: reference to function c(){},
  d: reference to FunctionExpression "d"
}
```

## 执行阶段
在代码执行阶段，会顺序执行代码，根据代码，修改变量对象的值
```js
AO = {
  arguments: {
    0: 1,
    length: 1
  },
  a: 1,
  b: 3,
  c: reference to function c(){},
  d: undefined
}
```

# 提升
变量和函数声明被提升到函数作用域的顶部
```js
console.log(typeof foo); // 函数指针
console.log(typeof bar); // undefined

var foo = 'hello',
bar = function() {
  return 'world';
};

function foo() {
  return 'hello';
}
```
- `foo`在代码执行阶段之前就已经在变量对象中被定义了
- `bar`实际上是一个变量，变量的值是函数，变量虽然在扫描变量声明阶段创建但他们被初始化为undefined。
