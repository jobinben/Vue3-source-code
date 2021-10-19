# Vue3的版本
    "version": "3.2.20"

# 阅读区域

    1700 - 1900

# 解决疑惑

    1. v-for的key的diff算法
    2. 为什么v-for需要加key

## 1. v-for的key的diff算法

在源码中，我们可以看到两个函数分别: `patchKeyedChildren()` 和 `patchUnkeyedChildren()`

从函数的名字就可以看出，一个处理没有key的时候，一个是处理有key的时候。

先来看一下Vue官方给出的说明:
1. key 特殊 attribute 主要用做 Vue 的虚拟 DOM 算法的提示，以在比对新旧节点组时辨识 VNodes。
2. 不使用 key时，Vue 会使用一种算法来最小化元素的移动并且尽可能`尝试就地修改/复用`相同类型元素。
3. 使用 key 时，它会基于 key 的顺序变化重新`排列元素`，并且那些使用了已经不存在的 key 的元素将会被`移除/销毁`。
   



## 2. 为什么v-for需要加key

因为不使用 `key`时，`Vue` 会使用一种算法来最小化元素的移动并且尽可能`尝试就地修改/复用`相同类型元素。
使用`key`的时候，相对于不使用`key`时，元素的`增删性能要好`。

（1）没有使用key时，`patchUnkeyedChildren()`函数源码
```ts
// 对v-for 如果没有key的diff算法
  const patchUnkeyedChildren = (
    c1: VNode[],
    c2: VNodeArrayChildren,
    container: RendererElement,
    anchor: RendererNode | null,
    parentComponent: ComponentInternalInstance | null,
    parentSuspense: SuspenseBoundary | null,
    isSVG: boolean,
    slotScopeIds: string[] | null,
    optimized: boolean
  ) => {
    c1 = c1 || EMPTY_ARR
    c2 = c2 || EMPTY_ARR
    // 获取旧结点树的长度
    const oldLength = c1.length
    // 获取新结点树的长度
    const newLength = c2.length
    // 取最小长度
    const commonLength = Math.min(oldLength, newLength)
    let i
    // 遍历结点(对VNode Tree的遍历对比的diff算法)
    for (i = 0; i < commonLength; i++) {
      const nextChild = (c2[i] = optimized
        ? cloneIfMounted(c2[i] as VNode)
        : normalizeVNode(c2[i]))
      // 进行新旧结点比对patch的算法
      patch( //传入参数
        c1[i], // 当前旧结点
        nextChild, // 新结点
        container,
        null,
        parentComponent,
        parentSuspense,
        isSVG,
        slotScopeIds,
        optimized
      )
    }

    // 当 旧结结点树的长度 大于新结点树的长度
    // 说明删除了结点，进行卸载旧结点
    if (oldLength > newLength) {
      // remove old
      unmountChildren(
        c1,
        parentComponent,
        parentSuspense,
        true,
        false,
        commonLength
      )
    } else {
      // 否则挂载新的结点
      // mount new
      mountChildren(
        c2,
        container,
        anchor,
        parentComponent,
        parentSuspense,
        isSVG,
        slotScopeIds,
        optimized,
        commonLength
      )
    }
  }
```

（2）使用key时，`patchKeyedChildren()`函数的源码

```ts
// v-for拥有key的diff算法
  const patchKeyedChildren = (
    c1: VNode[], // 旧结点树（旧的VNode Tree），也就是旧的虚拟DOM。
    c2: VNodeArrayChildren, // 新结点，也就是新的虚拟DOM，也就是新的一些VNode虚拟结点
    container: RendererElement,
    parentAnchor: RendererNode | null,
    parentComponent: ComponentInternalInstance | null,
    parentSuspense: SuspenseBoundary | null,
    isSVG: boolean,
    slotScopeIds: string[] | null,
    optimized: boolean
  ) => {
    let i = 0
    const l2 = c2.length
    // 旧虚拟DOM中的虚拟结点Vnode的个数
    let e1 = c1.length - 1 // prev ending index
    // 新结点VNodes长度
    let e2 = l2 - 1 // next ending index

    // 1. sync from start
    // (a b) c
    // (a b) d e
    // 从头结点开始同步遍历对比patch
    while (i <= e1 && i <= e2) {
      const n1 = c1[i]
      const n2 = (c2[i] = optimized
        ? cloneIfMounted(c2[i] as VNode)
        : normalizeVNode(c2[i]))
        // 当两个结点类型相同时，进行patch
      if (isSameVNodeType(n1, n2)) {
        patch(
          n1,
          n2,
          container,
          null,
          parentComponent,
          parentSuspense,
          isSVG,
          slotScopeIds,
          optimized
        )
      } else {
        // 如果不同就退出循环
        break
      }
      i++
    }

    // 2. sync from end
    // a (b c)
    // d e (b c)
    // 从尾结点开始同步遍历对比patch
    while (i <= e1 && i <= e2) {
      const n1 = c1[e1]
      const n2 = (c2[e2] = optimized
        ? cloneIfMounted(c2[e2] as VNode)
        : normalizeVNode(c2[e2]))
        // 比较新旧结点类型
      if (isSameVNodeType(n1, n2)) {
        patch(
          n1,
          n2,
          container,
          null,
          parentComponent,
          parentSuspense,
          isSVG,
          slotScopeIds,
          optimized
        )
      } else {
        // 结点类型不同退出循环
        break
      }
      e1--
      e2--
    }

    // 3. common sequence + mount
    // (a b)
    // (a b) c
    // i = 2, e1 = 1, e2 = 2
    // (a b)
    // c (a b)
    // i = 0, e1 = -1, e2 = 0
    if (i > e1) {
      if (i <= e2) {
        const nextPos = e2 + 1
        const anchor = nextPos < l2 ? (c2[nextPos] as VNode).el : parentAnchor
        while (i <= e2) {
          patch(
            null,
            (c2[i] = optimized
              ? cloneIfMounted(c2[i] as VNode)
              : normalizeVNode(c2[i])),
            container,
            anchor,
            parentComponent,
            parentSuspense,
            isSVG,
            slotScopeIds,
            optimized
          )
          i++
        }
      }
    }

    // 4. common sequence + unmount
    // (a b) c
    // (a b)
    // i = 2, e1 = 2, e2 = 1
    // a (b c)
    // (b c)
    // i = 0, e1 = 0, e2 = -1
    else if (i > e2) {
      while (i <= e1) {
        unmount(c1[i], parentComponent, parentSuspense, true)
        i++
      }
    }

    // 5. unknown sequence
    // [i ... e1 + 1]: a b [c d e] f g
    // [i ... e2 + 1]: a b [e d c h] f g
    // i = 2, e1 = 4, e2 = 5
    // 序列混乱的处理
    else {
      const s1 = i // prev starting index
      const s2 = i // next starting index

      // 5.1 build key:index map for newChildren
      const keyToNewIndexMap: Map<string | number | symbol, number> = new Map()
      for (i = s2; i <= e2; i++) {
        const nextChild = (c2[i] = optimized
          ? cloneIfMounted(c2[i] as VNode)
          : normalizeVNode(c2[i]))
        if (nextChild.key != null) {
          if (__DEV__ && keyToNewIndexMap.has(nextChild.key)) {
            warn(
              `Duplicate keys found during update:`,
              JSON.stringify(nextChild.key),
              `Make sure keys are unique.`
            )
          }
          keyToNewIndexMap.set(nextChild.key, i)
        }
      }
      
```