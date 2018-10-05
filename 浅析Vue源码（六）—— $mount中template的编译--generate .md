上两文章

[浅析Vue源码（四）—— $mount中template的编译--parse](https://juejin.im/post/5bb702b6e51d450e46286608)

[浅析Vue源码（五）—— $mount中template的编译--optimize](https://juejin.im/post/5bb7158c6fb9a05d12281ede)

parse，optimize函数的功能，这里，我们主要介绍generate。

generate 函数主要功能就是根据 AST 结构拼接生成 render function 的字符串。


```
export function generate (
  ast: ASTElement | void,
  options: CompilerOptions
): CodegenResult {
  const state = new CodegenState(options)
  const code = ast ? genElement(ast, state) : '_c("div")'
  return {
  // 最外层包一个 with(this) 之后返回
    render: `with(this){return ${code}}`,
    // 这个数组中的函数与 VDOM 中的 diff 算法优化相关
    // 我们会在编译阶段给后面不会发生变化的 VNode 节点打上 staticRoot 为 true 的标
    // 那些被标记为 staticRoot 节点的 VNode 就会单独生成 staticRenderFns
    staticRenderFns: state.staticRenderFns
  }
}
```
其中 genElement 函数是什么呢？--是会根据 AST 的属性调用不同的方法生成字符串返回。也就是拼接字符串了：

```
export function genElement (el: ASTElement, state: CodegenState): string {
  if (el.staticRoot && !el.staticProcessed) {
    return genStatic(el, state)
  } else if (el.once && !el.onceProcessed) {
    return genOnce(el, state)
  } else if (el.for && !el.forProcessed) {
    return genFor(el, state)
  } else if (el.if && !el.ifProcessed) {
    return genIf(el, state)
  } else if (el.tag === 'template' && !el.slotTarget) {
    return genChildren(el, state) || 'void 0'
  } else if (el.tag === 'slot') {
    return genSlot(el, state)
  } else {
    // component or element
    let code
    if (el.component) {
      code = genComponent(el.component, el, state)
    } else {
      const data = el.plain ? undefined : genData(el, state)

      const children = el.inlineTemplate ? null : genChildren(el, state, true)
      code = `_c('${el.tag}'${
        data ? `,${data}` : '' // data
      }${
        children ? `,${children}` : '' // children
      })`
    }
    // module transforms
    for (let i = 0; i < state.transforms.length; i++) {
      code = state.transforms[i](el, code)
    }
    return code
  }
}
```

以上就是 compile 函数中三个核心步骤的介绍，compile 之后我们得到了 render function 的字符串形式，后面通过 new Function 得到真正的渲染函数。DOM 初始化过程最后一步是根据渲染函数生成 Vnode，根据此 Vnode 生成真实 DOM，插入 DOM 树中，并将该 Vnode 记录为 preVnode。

要是喜欢就给我一个start，[github](https://github.com/DIVIBEAR/vue)

感谢[muwoo](https://github.com/muwoo/blogs/blob/master/src/Vue/9.md)提供的思路。