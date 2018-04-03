# 继承
# 1.原型链继承
```js
function Parent () {
  this.name = 'Kevin';
}

Parent.prototype.getName = function () {
  console.log(this.name);
}

function Child () {

}

Child.prototype = new Parent();
var child1 = new Child();
console.log(child1.getName())

// 问题
// 1. 引用类型的属性被所有实例共享
// 2. 在创建 Child 的实例时，不能向Parent传参
```
## 2. 构造函数
```js
function Parent () {
  this.names = ['kevin', 'daisy'];
}

function Child () {
  Parent.call(this);
}

var child1 = new Child();
child1.names.push('yayu');
console.log(child1.names); // ["kevin", "daisy", "yayu"]
var child2 = new Child();
console.log(child2.names); // ["kevin", "daisy"]

// 问题
// 1. 方法都在构造函数中定义，每次创建实例都会创建一遍方法。
```

## 3. 组合继承
```js
function Parent (name) {
  this.name = name;
  this.colors = ['red', 'blue', 'green'];
}

Parent.prototype.getName = function () {
  console.log(this.name)
}

function Child (name, age) {
  Parent.call(this, name);
  this.age = age;
}

Child.prototype = new Parent();
var child1 = new Child('kevin', '18');

child1.colors.push('black');

console.log(child1.name); // kevin
console.log(child1.age); // 18
console.log(child1.colors); // ["red", "blue", "green", "black"]

var child2 = new Child('daisy', '20');

console.log(child2.name); // daisy
console.log(child2.age); // 20
console.log(child2.colors); // ["red", "blue", "green"]

// 问题
// 1. 会调用多次父类的构造函数
```

## 4. 原型式继承
```js
function createObj(o) {
  function F(){}
  F.prototype = o;
  return new F();
}

// 问题
// 1. 包含引用类型的属性值始终都会共享相应的值，这点跟原型链继承一样。
```

## 5. 寄生式继承
```js
function createObj (o) {
  var clone = object.create(o);
  clone.sayName = function () {
    console.log('hi');
  }
  return clone;
}

// 问题
// 1. 跟借用构造函数模式一样，每次创建对象都会创建一遍方法。
```
## 6. 寄生组合式继承 (最优)
```js
function object(o) {
  function F() {}
  F.prototype = o;
  return new F();
}

function prototype(child, parent) {
  var prototype = object(parent.prototype);
  prototype.constructor = child;
  child.prototype = prototype;
}

prototype(Child, Parent);
```
