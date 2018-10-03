在上一篇文章《浅析Vue源码（一）--  造物创世》中提到在定义了一个 Vue Class的时候会引入initMixin 来初始化一些功能，这篇文章就来讲讲initMixin到底初始化了哪些原理呢？

initMixin来源init.js

```
let uid = 0

export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    // a uid
    vm._uid = uid++

    let startTag, endTag
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      startTag = `vue-perf-start:${vm._uid}`
      endTag = `vue-perf-end:${vm._uid}`
      mark(startTag)
    }
    // 如果是Vue的实例，则不需要被observe
    // a flag to avoid this being observed
    vm._isVue = true
    // merge options
    // 第一步： options参数的处理
    if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      // mergeOptions接下来我们会详细讲哦~
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    // 第二步： renderProxy
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }
    // expose real self
    vm._self = vm
    // 第三步： vm的生命周期相关变量初始化
    initLifecycle(vm)
    
    // 第四步： vm的事件监听初始化
    initEvents(vm)
    // 第五步： vm的编译render初始化
    initRender(vm)
    // 第六步： vm的beforeCreate生命钩子的回调
    callHook(vm, 'beforeCreate')
    // 第七步： vm在data/props初始化之前要进行绑定
    initInjections(vm) // resolve injections before data/props
    
    // 第八步： vm的sate状态初始化
    initState(vm)
    // 第九步： vm在data/props之后要进行提供
    initProvide(vm) // resolve provide after data/props
    // 第十步： vm的created生命钩子的回调
    callHook(vm, 'created')

    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      vm._name = formatComponentName(vm, false)
      mark(endTag)
      measure(`vue ${vm._name} init`, startTag, endTag)
    }
    // 第十一步：render & mount
    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```

从上面一点一点注释可以看出，主要是为我们的Vue原型上定义一个方法_init。然后当我们执行new Vue(options) 的时候，会调用这个方法。而这个_init方法的实现，便是我们需要关注的地方。 前面定义vm实例都挺好理解的，主要我们来看一下mergeOptions这个方法，其实Vue在实例化的过程中，会在代码运行后增加很多新的东西进去。我们把我们传入的这个对象叫options，实例中我们可以通过vm.$options访问到。

## mergeOptions
mergeOptions主要分成两块，就是resolveConstructorOptions(vm.constructor)和options，mergeOptions这个函数的功能就是要把这两个合在一起。options是我们通过new Vue(options)实例化传入的，所以，我们主要需要研究的是resolveConstructorOptions这个函数的功能。

### resolveConstructorOptions 处理 options

```
export function resolveConstructorOptions (Ctor: Class<Component>) {
  let options = Ctor.options
  // 首先需要判断该类是否是Vue的子类
  if (Ctor.super) {
    const superOptions = resolveConstructorOptions(Ctor.super)
    const cachedSuperOptions = Ctor.superOptions
    // 来判断父类中的options 有没有发生变化
    if (superOptions !== cachedSuperOptions) {
      // super option changed,
      // need to resolve new options.
      Ctor.superOptions = superOptions
      // check if there are any late-modified/attached options (#4976)
      const modifiedOptions = resolveModifiedOptions(Ctor)
      // update base extend options
      if (modifiedOptions) {
        // 当为Vue混入一些options时，superOptions会发生变化，此时于之前子类中存储的cachedSuperOptions已经不相等，所以下面的操作主要就是更新sub.superOptions
        extend(Ctor.extendOptions, modifiedOptions)
      }
      options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions)
      if (options.name) {
        options.components[options.name] = Ctor
      }
    }
  }
  return options
}
```
在这里我们大概会产生一个疑惑，可能不太清楚Ctor类里面每个属性到底代表了什么？
下面我们来看这一段代码：

```
 Sub.options = mergeOptions(
    Super.options,
    extendOptions
  )
  Sub['super'] = Super
  Sub.superOptions = Super.options
  Sub.extendOptions = extendOptions
  Sub.sealedOptions = extend({}, Sub.options)
```
继承自Super 的子类Sub。换句话说，Ctor其实是继承了Vue，是Vue的子类。
那又有一个疑惑了，mergeOptions是什么呢？

```
export function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  //...

  // 统一props格式
  normalizeProps(child)
  // 统一directives的格式
  normalizeDirectives(child)

  // 如果存在child.extends
  // ...
  // 如果存在child.mixins
  // ...

  // 针对不同的键值，采用不同的merge策略
  const options = {}
  let key
  for (key in parent) {
    mergeField(key)
  }
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  return options
}
```
上面采取了对不同的field采取不同的策略，Vue提供了一个strats对象，其本身就是一个hook,如果strats有提供特殊的逻辑，就走strats,否则走默认merge逻辑。

```
const strats = config.optionMergeStrategies
strats.el  = strats.propsData = ...
strats.data = ...
strats.watch  ...
....

const defaultStrat = function (parentVal: any, childVal: any): any {
  return childVal === undefined
    ? parentVal
    : childVal
}
```
看到这里，我们终于把业务逻辑以及组件的一些特性全都放到了vm.$options中了，后续的操作我们都可以从vm.$options拿到可用的信息。框架基本上都是对输入宽松，对输出严格，vue也是如此，不管使用者添加了什么代码，最后都规范的收入vm.$options中。

### renderProxy
这一步比较简单，主要是定义了vm._renderProxy,这是后期为render做准备的，作用是在render中将this指向vm._renderProxy。一般而言，vm._renderProxy是等于vm的，但在开发环境，Vue动用了Proxy这个新API，有关Proxy，大家可以读读深入浅出ES6（十二）：代理 Proxies, 这里不再展开。

[github]()喜欢的话可以给我一个start哦~
接下来会涉及到以下解析哦

```
initLifecycle(vm)
initEvents(vm)
initRender(vm)
callHook(vm, 'beforeCreate')
initInjections(vm)
initState(vm)
initProvide(vm)
callHook(vm, 'created')
```
