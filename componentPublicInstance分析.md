# Vue3的版本
    "version": "3.2.20"

# 阅读区域

    292 - 327

# 解决疑惑

    1. 当`OptionApi`和 `CompositionApi`同时存在时先执行哪个
    2. 同时定义一个变量，data(){}存放，setup(){}存放，最终变量取哪里的值

# 代码

```ts 
    let normalizedProps
    // key不是以$开头的属性时 （vm.$data、vm.$props、vm.$el、vm.$options、vm.$parent、vm.$root、vm.$children、vm.$slots、vm.$scopedSlots、vm.$refs、vm.$isServer、vm.$attrs、vm.$listeners）
    if (key[0] !== '$') {
      const n = accessCache![key]
      if (n !== undefined) {
        switch (n) {
            // 1. 先判断属性是不是setup
          case AccessTypes.SETUP:
            // 2. 如果是就直接返回setup中的属性 
            return setupState[key]
            // 3. 否则再判断data属性
          case AccessTypes.DATA:
            return data[key]
          case AccessTypes.CONTEXT:
        //   ctx属性的话指这些属性(method/computed里面的属性)
            return ctx[key]
          case AccessTypes.PROPS:
            return props![key]
          // default: just fallthrough
        }
      } else if (setupState !== EMPTY_OBJ && hasOwn(setupState, key)) {
        //   4.也是先判断是否存在setup属性
        accessCache![key] = AccessTypes.SETUP
        //  5. 然后进行返回
        return setupState[key]
      } else if (data !== EMPTY_OBJ && hasOwn(data, key)) {
        accessCache![key] = AccessTypes.DATA
        return data[key]
      } else if (
        // only cache other properties when instance has declared (thus stable)
        // props
        (normalizedProps = instance.propsOptions[0]) &&
        hasOwn(normalizedProps, key)
      ) {
        accessCache![key] = AccessTypes.PROPS
        return props![key]
      } else if (ctx !== EMPTY_OBJ && hasOwn(ctx, key)) {
        accessCache![key] = AccessTypes.CONTEXT
        return ctx[key]
      } else if (!__FEATURE_OPTIONS_API__ || shouldCacheAccess) {
        accessCache![key] = AccessTypes.OTHER
      }
    }
```

## 1. 当`OptionApi`和 `CompositionApi`同时存在时先执行哪个
当两个api同时存在时，vue会先执行compositionapi，执行顺序如下:

**`setup => data => props => ctx`**

## 2. 同时定义一个变量，data(){}存放，setup(){}存放，最终变量取哪里的值

示例代码:

```html
<h2>{{name}}</h2>
```
```js
export default{
    data() {
        return {
            name: 'dabing'
        }
    },

    setup() {
        const name = ref('jobin')
        return {
            name
        }
    }
}

```

html渲染时得到的name将是`jobin`
```html
<h2>jobin</h2>
```


根据源码可得出:**最终变量直接取setup()里面的变量 然后返回直接结束**。