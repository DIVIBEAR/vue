[浅析Vue源码（四）—— $mount中template的编译--parse](https://juejin.im/post/5bb702b6e51d450e46286608)

[浅析Vue源码（五）—— $mount中template的编译--optimize](https://juejin.im/post/5bb7158c6fb9a05d12281ede)

[浅析Vue源码（六）—— $mount中template的编译--generate](https://juejin.im/post/5bb718bbf265da0ac8494a7b)

前面我们用三片文章介绍了compile解析template，完成了 template --> AST --> render function 的过程。这篇我们主要介绍一下VNode的生成过程，在此之前，我们先来简单的了解一下什么是VNode？

先来看一下Vue.js源码中对VNode类的定义。

```
src/core/vdom/vnode.js
```

```
export default class VNode {
  tag: string | void;
  data: VNodeData | void;
  children: ?Array<VNode>;
  text: string | void;
  elm: Node | void;
  ns: string | void;
  context: Component | void; // rendered in this component's scope
  key: string | number | void;
  componentOptions: VNodeComponentOptions | void;
  componentInstance: Component | void; // component instance
  parent: VNode | void; // component placeholder node

  // strictly internal
  raw: boolean; // contains raw HTML? (server only)
  isStatic: boolean; // hoisted static node
  isRootInsert: boolean; // necessary for enter transition check
  isComment: boolean; // empty comment placeholder?
  isCloned: boolean; // is a cloned node?
  isOnce: boolean; // is a v-once node?
  asyncFactory: Function | void; // async component factory function
  asyncMeta: Object | void;
  isAsyncPlaceholder: boolean;
  ssrContext: Object | void;
  fnContext: Component | void; // real context vm for functional nodes
  fnOptions: ?ComponentOptions; // for SSR caching
  fnScopeId: ?string; // functional scope id support

  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions,
    asyncFactory?: Function
  ) {
    /*当前节点的标签名*/
    this.tag = tag
    /*当前节点对应的对象，包含了具体的一些数据信息，是一个VNodeData类型，可以参考VNodeData类型中的数据信息*/
    this.data = data
    /*当前节点的子节点，是一个数组*/
    this.children = children
    /*当前节点的文本*/
    this.text = text
    /*当前虚拟节点对应的真实dom节点*/
    this.elm = elm
    /*当前节点的名字空间*/
    this.ns = undefined
    /*编译作用域*/
    this.context = context
    /*函数化组件作用域*/
    this.functionalContext = undefined
    /*节点的key属性，被当作节点的标志，用以优化*/
    this.key = data && data.key
    /*组件的option选项*/
    this.componentOptions = componentOptions
    /*当前节点对应的组件的实例*/
    this.componentInstance = undefined
    /*当前节点的父节点*/
    this.parent = undefined
    /*简而言之就是是否为原生HTML或只是普通文本，innerHTML的时候为true，textContent的时候为false*/
    this.raw = false
    /*静态节点标志*/
    this.isStatic = false
    /*是否作为跟节点插入*/
    this.isRootInsert = true
    /*是否为注释节点*/
    this.isComment = false
    /*是否为克隆节点*/
    this.isCloned = false
    /*是否有v-once指令*/
    this.isOnce = false
    /*异步组件的工厂方法*/
    this.asyncFactory = asyncFactory
    /*异步源*/
    this.asyncMeta = undefined
    /*是否异步的预赋值*/
    this.isAsyncPlaceholder = false
  }

  // DEPRECATED: alias for componentInstance for backwards compat.
  /* istanbul ignore next */
  get child (): Component | void {
    return this.componentInstance
  }
}
```
这是一个最基础的VNode节点，作为其他派生VNode类的基类，里面定义了下面这些数据。

tag: 当前节点的标签名

data: 当前节点对应的对象，包含了具体的一些数据信息，是一个VNodeData类型，可以参考VNodeData类型中的数据信息

children: 当前节点的子节点，是一个数组

text: 当前节点的文本

elm: 当前虚拟节点对应的真实dom节点

ns: 当前节点的名字空间

context: 当前节点的编译作用域

functionalContext: 函数化组件作用域

key: 节点的key属性，被当作节点的标志，用以优化

componentOptions: 组件的option选项

componentInstance: 当前节点对应的组件的实例

parent: 当前节点的父节点

raw: 简而言之就是是否为原生HTML或只是普通文本，innerHTML的时候为true，textContent的时候为false

isStatic: 是否为静态节点

isRootInsert: 是否作为跟节点插入

isComment: 是否为注释节点

isCloned: 是否为克隆节点

isOnce: 是否有v-once指令

asyncFactory：异步组件的工厂方法

asyncMeta：异步源

isAsyncPlaceholder：是否异步的预赋值

包含了我们常用的一些标记节点的属性，为什么是VNode？其实网上已经有大量的资料可以了解到VNode的优势，比如大量副交互的性能优势，ssr的支持，跨平台Weex的支持...这里不再赘述。我们接下来看一下VNode的基本分类： 

![](https://user-gold-cdn.xitu.io/2018/10/6/166482d1633e4761?w=495&h=540&f=png&s=64110)

我们首先来看一个例子：

```
{
    tag: 'div'
    data: {
        class: 'test'
    },
    children: [
        {
            tag: 'span',
            data: {
                class: 'demo'
            }
            text: 'hello,VNode'
        }
    ]
}
```
渲染之后的结果就是这样的

```
<div class="test">
    <span class="demo">hello,VNode</span>
</div>
```
## 生成 VNode
有了之前章节的知识，现在我们需要编译下面这段template

```
<div id="app">
  <header>
    <h1>I'm a template!</h1>
  </header>
  <p v-if="message">
    {{ message }}
  </p>
  <p v-else>
    No message.
  </p>
</div>
```
得到这样的render function

```
(function() {
  with(this){
    return _c('div',{   //创建一个 div 元素
      attrs:{"id":"app"}  //div 添加属性 id
      },[
        _m(0),  //静态节点 header，此处对应 staticRenderFns 数组索引为 0 的 render function
        _v(" "), //空的文本节点
        (message) //三元表达式，判断 message 是否存在
         //如果存在，创建 p 元素，元素里面有文本，值为 toString(message)
        ?_c('p',[_v("\n    "+_s(message)+"\n  ")])
        //如果不存在，创建 p 元素，元素里面有文本，值为 No message.
        :_c('p',[_v("\n    No message.\n  ")])
      ]
    )
  }
})
```
要看懂上面的 render function，只需要了解 _c，_m，_v，_s这几个函数的定义，其中 _c 是 createElement，_m 是 renderStatic，_v 是 createTextVNode，_s 是 toString。 我们在编译的过程中调用了vm._render() 方法，其中_render函数中有这样一句：

```
const { render, _parentVnode } = vm.$options
...
vnode = render.call(vm._renderProxy, vm.$createElement)
```
接下来我们系统的讲一个如何创建Vnode。
### 生成一个新的VNode的方法

#### createEmptyVNode 创建一个空VNode节点
```
/*创建一个空VNode节点*/
export const createEmptyVNode = () => {
  const node = new VNode()
  node.text = ''
  node.isComment = true
  return node
}
```
#### createTextVNode 创建一个文本节点

```
/*创建一个文本节点*/
export function createTextVNode (val: string | number) {
  return new VNode(undefined, undefined, undefined, String(val))
}
```
#### createComponent 创建一个组件节点

```
src/core/vdom/create-element.js
```

```
// wrapper function for providing a more flexible interface
// without getting yelled at by flow
export function createElement (
  context: Component,
  tag: any,
  data: any,
  children: any,
  normalizationType: any,
  alwaysNormalize: boolean
): VNode | Array<VNode> {
/*兼容不传data的情况*/
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children
    children = data
    data = undefined
  }
  /*如果alwaysNormalize为true，则normalizationType标记为ALWAYS_NORMALIZE*/
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE
  }
  /*创建虚拟节点*/
  return _createElement(context, tag, data, children, normalizationType)
}
/*创建虚拟节点*/
export function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
/*
    如果传递data参数且data的__ob__已经定义（代表已经被observed，上面绑定了Oberver对象），
    https://cn.vuejs.org/v2/guide/render-function.html#约束
    那么创建一个空节点
  */
  if (isDef(data) && isDef((data: any).__ob__)) {
    process.env.NODE_ENV !== 'production' && warn(
      `Avoid using observed data object as vnode data: ${JSON.stringify(data)}\n` +
      'Always create fresh vnode data objects in each render!',
      context
    )
    return createEmptyVNode()
  }
  // object syntax in v-bind
  if (isDef(data) && isDef(data.is)) {
    tag = data.is
  }
  /*如果tag不存在也是创建一个空节点*/
  if (!tag) {
    // in case of component :is set to falsy value
    return createEmptyVNode()
  }
  // warn against non-primitive key
  if (process.env.NODE_ENV !== 'production' &&
    isDef(data) && isDef(data.key) && !isPrimitive(data.key)
  ) {
    if (!__WEEX__ || !('@binding' in data.key)) {
      warn(
        'Avoid using non-primitive value as key, ' +
        'use string/number value instead.',
        context
      )
    }
  }
  // support single function children as default scoped slot
  /*默认作用域插槽*/
  if (Array.isArray(children) &&
    typeof children[0] === 'function'
  ) {
    data = data || {}
    data.scopedSlots = { default: children[0] }
    children.length = 0
  }
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
  let vnode, ns
  if (typeof tag === 'string') {
    let Ctor 
    /*获取tag的名字空间*/
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    /*判断是否是保留的标签*/
    if (config.isReservedTag(tag)) {
      // platform built-in elements
       /*如果是保留的标签则创建一个相应节点*/
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if (isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      /*从vm实例的option的components中寻找该tag，存在则就是一个组件，创建相应节点，Ctor为组件的构造类*/
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // unknown or unlisted namespaced elements
      // check at runtime because it may get assigned a namespace when its
      // parent normalizes children
      /*未知的元素，在运行时检查，因为父组件可能在序列化子组件的时候分配一个名字空间*/
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
     /*tag不是字符串的时候则是组件的构造类*/
    vnode = createComponent(tag, data, context, children)
  }
  if (Array.isArray(vnode)) {
    return vnode
  } else if (isDef(vnode)) {
  /*如果有名字空间，则递归所有子节点应用该名字空间*/
    if (isDef(ns)) applyNS(vnode, ns)
    if (isDef(data)) registerDeepBindings(data)
    return vnode
  } else {
  /*如果vnode没有成功创建则创建空节点*/
    return createEmptyVNode()
  }
}
```
createElement用来创建一个虚拟节点。当data上已经绑定__ob__的时候，代表该对象已经被Oberver过了，所以创建一个空节点。tag不存在的时候同样创建一个空节点。当tag不是一个String类型的时候代表tag是一个组件的构造类，直接用new VNode创建。当tag是String类型的时候，如果是保留标签，则用new VNode创建一个VNode实例，如果在vm的option的components找得到该tag，代表这是一个组件，否则统一用new VNode创建。

要是喜欢就给我一个star，[github](https://github.com/DIVIBEAR/vue)