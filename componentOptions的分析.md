# Vue3的版本
    "version": "3.2.20"

# 阅读区域

    618 - 645

# 解决疑惑

    1. methods 对象的 this指向
    2. methods 对象的里的 方法 为什么不能用箭头函数

## 代码

``` ts
// 当 methods 有方法时
if (methods) {
    // 对methods对象中的每一个方法遍历
    for (const key in methods) {
        // 取出每一个方法
      const methodHandler = (methods as MethodOptions)[key]
        // 判断是否是方法Function类型
      if (isFunction(methodHandler)) {
        // In dev mode, we use the `createRenderContext` function to define
        // methods to the proxy target, and those are read-only but
        // reconfigurable, so it needs to be redefined here
        // 在开发环境时
        if (__DEV__) {
          Object.defineProperty(ctx, key, {
            // 对每个方法进行绑定this的作用域 this等于这里的publicThis
            value: methodHandler.bind(publicThis),
            configurable: true,
            enumerable: true,
            writable: true
          })
        } else {
            // 非开发环境时，对每个方法进行this的绑定
          ctx[key] = methodHandler.bind(publicThis)
        }
        if (__DEV__) {
          checkDuplicateProperties!(OptionTypes.METHODS, key)
        }
      } else if (__DEV__) {
        warn(
          `Method "${key}" has type "${typeof methodHandler}" in the component definition. ` +
            `Did you reference the function correctly?`
        )
      }
    }
  }

```

## 1. methods 对象的 this指向

从上面的源码看出，我们找到 publicThis的指向是 `实例的代理`

```ts
const publicThis = instance.proxy! as any
```


## 2. methods 对象的里的 方法 为什么不能用箭头函数

 **箭头函数表达式** 的语法比函数表达式更简洁，并且没有自己的`this`，`arguments`，`super`或`new.target`

 如果你用箭头表达式，那么它的this将为指向window

理解this实例
 ```js
'use strict';
var obj = {
  i: 10,
  b: () => console.log(this.i, this),
  c: function() {
    console.log( this.i, this)
  }
}
obj.b();
// undefined, Window{...}
obj.c();
// 10, Object {...}

 ```

### 分析绑定时，用bind绑定后的结果

 ```js
let obj = {
  a: 10
};
let fun = () => {
    console.log(this.a, typeof this.a, this);
    return this.a+10;
   // 代表全局对象 'Window', 因此 'this.a' 返回 'undefined'
}
Object.defineProperty(obj, "b", {
    get: fun.bind(obj) //尝试改变this的指向
});

obj.b; // undefined   "undefined"   Window {postMessage: ƒ, blur: ƒ, focus: ƒ, close: ƒ, frames: Window, …}

 ```

 现在我相信你可以理解在methods里不能用箭头函数的原因啦。

