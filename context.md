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
1. 创建变量对象
  + 创建arguments
  + 扫描函数声明
  + 扫描变量声明
2. 初始化作用域链
3. 求this
## 执行阶段
1. 初始化变量和函数的引用
2. 执行代码


# 创建变量对象
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

# 作用域链 (Scope chain)
当查找变量的时候，会先从当前上下文的变量对象中查找，如果没有找到，就会从父级(词法层面上的父级)执行上下文的变量对象中查找，一直找到全局上下文的变量对象，也就是全局对象。这样由多个执行上下文的变量对象构成的链表就叫做作用域链。


# 函数创建
函数的作用域在函数定义的时候就决定了</br>

这是因为函数有一个内部属性 `[[scope]]`，当函数创建的时候，就会保存所有父变量对象到其中，你可以理解 `[[scope]]` 就是所有父变量对象的层级链，但是注意：`[[scope]]` 并不代表完整的作用域链！</br>
```js
function foo() {
  function bar() {
    
  }
}
```
```js
foo.[[scope]] = [
  globalContext.VO
];

bar.[[scope]] = [
  fooContext.AO
  globalContext.VO
];
```
# 函数激活
进入函数上下文，创建 VO/AO 后，就会将活动对象添加到作用链的前端
`Scope = [AO].concat([[Scope]]);`

# 流程
```js
var scope = "global scope";
function checkscope(){
  var scope2 = 'local scope';
  return scope2;
}
checkscope();
```
1. checkscope 函数被创建，保存作用域链到 内部属性`[[scope]]`
```js
checkscope.[[scope]] = [
  globalContext.VO
]
```
2. 执行 `checkscope` 函数，创建 `checkscope` `函数执行上下文，checkscope` 函数执行上下文被压入执行上下文栈
```js
ECStack = [
  checkscopeContext,
  globalContext
];
```
3. checkscope 函数并不立刻执行，开始做准备工作，第一步：复制函数[[scope]]属性创建作用域链
```js
checkscopeContext = {
  Scope: checkscope.[[scope]],
}
```
4. 第二步：用 `arguments` 创建活动对象，随后初始化活动对象，加入形参、函数声明、变量声明
```js
checkscopeContext = {
  Scope: checkscope.[[scope]],
  AO: {
    arguments: {
      length: 0
    },
    scope2: undefined
  }
}
```
5. 第三步：将活动对象压入 checkscope 作用域链顶端
```js
checkscopeContext = {
  Scope: [AO, [[scope]]],
  AO: {
    arguments: {
      length: 0
    },
    scope2: undefined
  }
}
```
6. 准备工作做完，开始执行函数，随着函数的执行，修改 AO 的属性值
```js
checkscopeContext = {
  Scope: [AO, [[scope]]],
  AO: {
    arguments: {
      length: 0
    },
    scope2: 'local scope'
  }
}
```
7. 查找到 scope2 的值，返回后函数执行完毕，函数上下文从执行上下文栈中弹出
```js
ECStack = [
  globalContext
];
```

# This
Reference 是一个 Specification Type，也就是 “只存在于规范里的抽象类型”。它们是为了更好地描述语言的底层行为逻辑才存在的，但并不存在于实际的 js 代码中</br>
Reference由三个组成部分，分别是：
- base value
- referenced name
- strict reference

base value 就是属性所在的对象或者就是 EnvironmentRecord, 它的值只可能是 undefined, an Object, a Boolean, a String, a Number, or an environment record 其中的一种。
```js
var foo = 1;
var fooReference = {
  base: EnvironmentRecord,
  name: 'foo',
  strict: false
}
```
```js
var foo = {
  bar: function () {
    return this;
  }
};

foo.bar(); // foo

// bar对应的Reference是：
var BarReference = {
  base: foo,
  propertyName: 'bar',
  strict: false
};
```

## 确定this的值
1. 计算 MemberExpression 的结果赋值给 ref
2. 判断 ref 是不是一个 Reference 类型
  + 2.1 如果 ref 是 Reference，并且 IsPropertyReference(ref) 是 true, 那么 this 的值为 GetBase(ref)
  + 2.2 如果 ref 是 Reference，并且 base value 值是 Environment Record, 那么this的值为 ImplicitThisValue(ref)
  + 2.3 如果 ref 不是 Reference，那么 this 的值为 undefined

## 具体分析
1. 计算 MemberExpression 的结果赋值给 ref
  MemberExpression
  - PrimaryExpression 原始表达式
  - FunctionExpression 函数定义表达式
  - MemberExpression [ Expression ] 属性访问表达式
  - MemberExpression . IdentifierName 属性访问表达式
  - new MemberExpression Arguments 对象创建表达式
```js
function foo() {
  console.log(this)
}

foo(); // MemberExpression 是 foo

function foo() {
  return function() {
    console.log(this)
  }
}

foo()(); // MemberExpression 是 foo()

var foo = {
  bar: function () {
    return this;
  }
}

foo.bar(); // MemberExpression 是 foo.bar
```
## foo.bar()
MemberExpression 计算的结果是 foo.bar, foo.bar 是一个 Reference
```js
var Reference = {
  base: foo,
  name: 'bar',
  strict: false
};
```
- IsPropertyReference 方法，如果 base value 是一个对象，结果返回 true。
- base value 为 foo，是一个对象，所以 IsPropertyReference(ref) 结果为 true。
`this = GetBase(ref)， this 也就是　foo`
