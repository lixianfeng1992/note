# 闭包
MDN 对闭包的定义为： 闭包是指那些能够访问自由变量的函数。 自由变量是指在函数中使用的，但既不是函数参数也不是函数的局部变量的变量

# 分析
```js
var scope = "global scope";
function checkscope(){
  var scope = "local scope";
  function f(){
    return scope;
  }
  return f;
}

var foo = checkscope();
foo();
```
## 闭包的执行过程
1. 进入全局代码，创建全局执行上下文，全局执行上下文压入执行上下文栈
2. 全局执行上下文初始化
3. 执行函数`checkscope` 
+ 3.1 创建上下文, 入栈
+ 3.2 执行上下文, 创建参数变量, 作用域链([AO, globalContext.VO]), this
+ 3.3 执行完毕, 上下文弹出
4. 执行函数`foo`
+ 4.1 创建`f`上下文, 入栈
+ 4.2 执行上下文, 创建参数变量, 作用域链([AO, checkscopeContext.AO, globalContext.VO]), this
+ 4.3 执行完毕, 上下文弹出

由于执行闭包函数`f`的时候, 作用域链为`[AO, checkscopeContext.AO, globalContext.VO]`, 当在AO中找不到变量 `scope` 的时候会去 `checkscopeContext.AO` 中寻找, 所以结果为 `local scope`
