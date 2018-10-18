近期被问到一个问题，在你们项目中使用的是Vue的SPA（单页面）还是Vue的多页面设计？

这篇文章主要围绕Vue的SPA单页面设计展开。
关于如何展开Vue多页面设计[请点击查看](https://juejin.im/post/5a8e3f00f265da4e747fc700)。

## vue-router是什么？
首先我们需要知道vue-router是什么，它是干什么的？

这里指的路由并不是指我们平时所说的硬件路由器，这里的路由就是SPA（单页应用）的路径管理器。
换句话说，vue-router就是WebApp的链接路径管理系统。

vue-router是Vue.js官方的路由插件，它和vue.js是深度集成的，适合用于构建单页面应用。

那与传统的页面跳转有什么区别呢？

1.vue的单页面应用是基于路由和组件的，路由用于设定访问路径，并将路径和组件映射起来。

2.传统的页面应用，是用一些超链接来实现页面切换和跳转的。

在vue-router单页面应用中，则是路径之间的切换，也就是组件的切换。路由模块的本质 就是建立起url和页面之间的映射关系。

至于为啥不能用a标签，这是因为用Vue做的都是单页应用，就相当于只有一个主的index.html页面，所以你写的<a></a>标签是不起作用的，必须使用vue-router来进行管理。

## vue-router实现原理
SPA(single page application):单一页面应用程序，有且只有一个完整的页面；当它在加载页面的时候，不会加载整个页面的内容，而只更新某个指定的容器中内容。

单页面应用(SPA)的核心之一是:

1.更新视图而不重新请求页面;

2.vue-router在实现单页面前端路由时，提供了三种方式：Hash模式、History模式、abstract模式，根据mode参数来决定采用哪一种方式。

#### 路由模式
vue-router 提供了三种运行模式：

● hash: 使用 URL hash 值来作路由。默认模式。

● history: 依赖 HTML5 History API 和服务器配置。查看 HTML5 History 模式。

● abstract: 支持所有 JavaScript 运行环境，如 Node.js 服务器端。

### Hash模式
**vue-router 默认模式是 hash 模式 —— 使用 URL 的 hash 来模拟一个完整的 URL，当 URL 改变时，页面不会去重新加载**。 

hash（#）是URL 的锚点，代表的是网页中的一个位置，单单改变#后的部分（/#/..），浏览器只会加载相应位置的内容，不会重新加载网页，也就是说 #是用来指导浏览器动作的，对服务器端完全无用，HTTP请求中不包括#；同时每一次改变#后的部分，都会在浏览器的访问历史中增加一个记录，使用”后退”按钮，就可以回到上一个位置；**所以说Hash模式通过锚点值的改变，根据不同的值，渲染指定DOM位置的不同数据**。

### History模式
HTML5 History API提供了一种功能，能让开发人员在不刷新整个页面的情况下修改站点的URL，就是利用 history.pushState API 来完成 URL 跳转而无须重新加载页面；

由于hash模式会在url中自带#，如果不想要很丑的 hash，我们可以用路由的 history 模式，只需要在配置路由规则时，加入"mode: 'history'",这种模式充分利用 history.pushState API 来完成 URL 跳转而无须重新加载页面。

```
//main.js文件中
const router = new VueRouter({
  mode: 'history',
  routes: [...]
})
```
当使用 history 模式时，URL 就像正常的 url，例如 http://yoursite.com/user/id，比较好看！
不过这种模式要玩好，还需要后台配置支持。因为我们的应用是个单页客户端应用，如果后台没有正确的配置，当用户在浏览器直接访问 

所以呢，你要在服务端增加一个覆盖所有情况的候选资源：如果 URL 匹配不到任何静态资源，则应该返回同一个 index.html 页面，这个页面就是你 app 依赖的页面。

```
export const routes = [ 
  {path: "/", name: "homeLink", component:Home}
  {path: "/register", name: "registerLink", component: Register},
  {path: "/login", name: "loginLink", component: Login},
  {path: "*", redirect: "/"}]
  
```
此处就设置如果URL输入错误或者是URL 匹配不到任何静态资源，就自动跳到到Home页面。

### abstract模式
abstract模式是使用一个不依赖于浏览器的浏览历史虚拟管理后端。

根据平台差异可以看出，在 Weex 环境中只支持使用 abstract 模式。 不过，vue-router 自身会对环境做校验，如果发现没有浏览器的 API，vue-router 会自动强制进入 abstract 模式，所以 在使用 vue-router 时只要不写 mode 配置即可，默认会在浏览器环境中使用 hash 模式，在移动端原生环境中使用 abstract 模式。 （当然，你也可以明确指定在所有情况下都使用 abstract 模式）

## vue-router使用方式
**1:下载** npm i vue-router -S

**2:在main.js中引入 ** import VueRouter from 'vue-router';

**3:安装插件** Vue.use(VueRouter);

**4:创建路由对象并配置路由规则**

let router = new VueRouter({routes:[{path:'/home',component:Home}]});

**5:将其路由对象传递给Vue的实例**，options中加入 router:router

**6:在app.vue中留坑** 
```
<router-view></router-view>
```

具体实现请看如下代码：

```
//main.js文件中引入
import Vue from 'vue';
import VueRouter from 'vue-router';
//主体
import App from './components/app.vue';
import index from './components/index.vue'
//安装插件
Vue.use(VueRouter); //挂载属性
//创建路由对象并配置路由规则
let router = new VueRouter({
    routes: [
        //一个个对象
        { path: '/index', component: index }
    ]
});
//new Vue 启动
new Vue({
    el: '#app',
    //让vue知道我们的路由规则
    router: router, //可以简写router
    render: c => c(App),
})
```
最后记得在在app.vue中“留坑”

```
//app.vue中
<template>
    <div>
        <!-- 留坑，非常重要 -->
        <router-view></router-view>
    </div>
</template>
<script>
    export default {
        data(){
            return {}
        }
    }
</script>
```
## vue-router源码分析
我们先来看看vue的实现路径。

![](https://user-gold-cdn.xitu.io/2018/10/18/16684f162d3090eb?w=1640&h=1180&f=png&s=255854)

在入口文件中需要实例化一个 VueRouter 的实例对象 ，然后将其传入 Vue 实例的 options 中。

```
export default class VueRouter {
  static install: () => void;
  static version: string;

  app: any;
  apps: Array<any>;
  ready: boolean;
  readyCbs: Array<Function>;
  options: RouterOptions;
  mode: string;
  history: HashHistory | HTML5History | AbstractHistory;
  matcher: Matcher;
  fallback: boolean;
  beforeHooks: Array<?NavigationGuard>;
  resolveHooks: Array<?NavigationGuard>;
  afterHooks: Array<?AfterNavigationHook>;

  constructor (options: RouterOptions = {}) {
    this.app = null
    this.apps = []
    this.options = options
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    // 创建 matcher 匹配函数
    this.matcher = createMatcher(options.routes || [], this)
    // 根据 mode 实例化具体的 History，默认为'hash'模式
    let mode = options.mode || 'hash'
    // 通过 supportsPushState 判断浏览器是否支持'history'模式
    // 如果设置的是'history'但是如果浏览器不支持的话，'history'模式会退回到'hash'模式
    // fallback 是当浏览器不支持 history.pushState 控制路由是否应该回退到 hash 模式。默认值为 true。
    this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
    if (this.fallback) {
      mode = 'hash'
    }
    // 不在浏览器内部的话，就会变成'abstract'模式
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode
     // 根据不同模式选择实例化对应的 History 类
    switch (mode) {
      case 'history':
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        this.history = new HashHistory(this, options.base, this.fallback)
        break
      case 'abstract':
        this.history = new AbstractHistory(this, options.base)
        break
      default:
        if (process.env.NODE_ENV !== 'production') {
          assert(false, `invalid mode: ${mode}`)
        }
    }
  }

  match (
    raw: RawLocation,
    current?: Route,
    redirectedFrom?: Location
  ): Route {
    return this.matcher.match(raw, current, redirectedFrom)
  }

  get currentRoute (): ?Route {
    return this.history && this.history.current
  }

  init (app: any /* Vue component instance */) {
    process.env.NODE_ENV !== 'production' && assert(
      install.installed,
      `not installed. Make sure to call \`Vue.use(VueRouter)\` ` +
      `before creating root instance.`
    )

    this.apps.push(app)

    // main app already initialized.
    if (this.app) {
      return
    }

    this.app = app

    const history = this.history
    // 根据history的类别执行相应的初始化操作和监听
    if (history instanceof HTML5History) {
      history.transitionTo(history.getCurrentLocation())
    } else if (history instanceof HashHistory) {
      const setupHashListener = () => {
        history.setupListeners()
      }
      history.transitionTo(
        history.getCurrentLocation(),
        setupHashListener,
        setupHashListener
      )
    }

    history.listen(route => {
      this.apps.forEach((app) => {
        app._route = route
      })
    })
  }
  // 路由跳转之前
  beforeEach (fn: Function): Function {
    return registerHook(this.beforeHooks, fn)
  }
  // 路由导航被确认之间前
  beforeResolve (fn: Function): Function {
    return registerHook(this.resolveHooks, fn)
  }
  // 路由跳转之后
  afterEach (fn: Function): Function {
    return registerHook(this.afterHooks, fn)
  }
  // 第一次路由跳转完成时被调用的回调函数
  onReady (cb: Function, errorCb?: Function) {
    this.history.onReady(cb, errorCb)
  }
  // 路由报错
  onError (errorCb: Function) {
    this.history.onError(errorCb)
  }
  // 路由添加，这个方法会向history栈添加一个记录，点击后退会返回到上一个页面。
  push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    this.history.push(location, onComplete, onAbort)
  }
  // 这个方法不会向history里面添加新的记录，点击返回，会跳转到上上一个页面。上一个记录是不存在的。
  replace (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    this.history.replace(location, onComplete, onAbort)
  }
  // 相对于当前页面向前或向后跳转多少个页面,类似 window.history.go(n)。n可为正数可为负数。正数返回上一个页面
  go (n: number) {
    this.history.go(n)
  }
  // 后退到上一个页面
  back () {
    this.go(-1)
  }
  // 前进到下一个页面
  forward () {
    this.go(1)
  }

  getMatchedComponents (to?: RawLocation | Route): Array<any> {
    const route: any = to
      ? to.matched
        ? to
        : this.resolve(to).route
      : this.currentRoute
    if (!route) {
      return []
    }
    return [].concat.apply([], route.matched.map(m => {
      return Object.keys(m.components).map(key => {
        return m.components[key]
      })
    }))
  }

  resolve (
    to: RawLocation,
    current?: Route,
    append?: boolean
  ): {
    location: Location,
    route: Route,
    href: string,
    // for backwards compat
    normalizedTo: Location,
    resolved: Route
  } {
    const location = normalizeLocation(
      to,
      current || this.history.current,
      append,
      this
    )
    const route = this.match(location, current)
    const fullPath = route.redirectedFrom || route.fullPath
    const base = this.history.base
    const href = createHref(base, fullPath, this.mode)
    return {
      location,
      route,
      href,
      // for backwards compat
      normalizedTo: location,
      resolved: route
    }
  }

  addRoutes (routes: Array<RouteConfig>) {
    this.matcher.addRoutes(routes)
    if (this.history.current !== START) {
      this.history.transitionTo(this.history.getCurrentLocation())
    }
  }
}
```

### HashHistory
• hash虽然出现在url中，但不会被包括在http请求中，它是用来指导浏览器动作的，对服务器端没影响，因此，改变hash不会重新加载页面。

• 可以为hash的改变添加监听事件：

```
window.addEventListener("hashchange",funcRef,false)
```
• 每一次改变hash(window.location.hash)，都会在浏览器访问历史中增加一个记录。


```
export class HashHistory extends History {
  constructor (router: Router, base: ?string, fallback: boolean) {
    super(router, base)
    // check history fallback deeplinking
    // 如果是从history模式降级来的，需要做降级检查
    if (fallback && checkFallback(this.base)) {
    // 如果降级且做了降级处理，则返回
      return
    }
    ensureSlash()
  }
  .......
```

```
function checkFallback (base) {
  const location = getLocation(base)
  // 得到除去base的真正的 location 值
  if (!/^\/#/.test(location)) {
  // 如果此时地址不是以 /# 开头的
  // 需要做一次降级处理，降为 hash 模式下应有的 /# 开头
    window.location.replace(
      cleanPath(base + '/#' + location)
    )
    return true
  }
}

function ensureSlash (): boolean {
// 得到 hash 值
  const path = getHash()
  if (path.charAt(0) === '/') {
   // 如果是以 / 开头的，直接返回即可
    return true
  }
  // 不是的话，需要手动保证一次 替换 hash 值
  replaceHash('/' + path)
  return false
}

export function getHash (): string {
  // We can't use window.location.hash here because it's not
  // consistent across browsers - Firefox will pre-decode it!
  // 因为兼容性的问题，这里没有直接使用 window.location.hash
  // 因为 Firefox decode hash 值
  const href = window.location.href
  const index = href.indexOf('#')
  return index === -1 ? '' : decodeURI(href.slice(index + 1))
}
// 得到hash之前的url地址
function getUrl (path) {
  const href = window.location.href
  const i = href.indexOf('#')
  const base = i >= 0 ? href.slice(0, i) : href
  return `${base}#${path}`
}
// 添加一个hash
function pushHash (path) {
  if (supportsPushState) {
    pushState(getUrl(path))
  } else {
    window.location.hash = path
  }
}
// 替代hash
function replaceHash (path) {
  if (supportsPushState) {
    replaceState(getUrl(path))
  } else {
    window.location.replace(getUrl(path))
  }
}
```
hash的改变会自动添加到浏览器的访问历史记录中。 那么视图的更新是怎么实现的呢，看下 transitionTo()方法：

```
transitionTo (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    const route = this.router.match(location, this.current) //找到匹配路由
    this.confirmTransition(route, () => { //确认是否转化
      this.updateRoute(route) //更新route
      onComplete && onComplete(route)
      this.ensureURL()

      // fire ready cbs once
      if (!this.ready) {
        this.ready = true
        this.readyCbs.forEach(cb => { cb(route) })
      }
    }, err => {
      if (onAbort) {
        onAbort(err)
      }
      if (err && !this.ready) {
        this.ready = true
        this.readyErrorCbs.forEach(cb => { cb(err) })
      }
    })
  }
  
//更新路由
updateRoute (route: Route) {
    const prev = this.current // 跳转前路由
    this.current = route // 装备跳转路由
    this.cb && this.cb(route) // 回调函数，这一步很重要，这个回调函数在index文件中注册，会更新被劫持的数据 _router
    this.router.afterHooks.forEach(hook => {
      hook && hook(route, prev)
    })
  }
}
```

#### pushState 

```
export function pushState (url?: string, replace?: boolean) {
  saveScrollPosition()
  // try...catch the pushState call to get around Safari
  // DOM Exception 18 where it limits to 100 pushState calls
  // 加了 try...catch 是因为 Safari 有调用 pushState 100 次限制
  // 一旦达到就会抛出 DOM Exception 18 错误
  const history = window.history
  try {
    if (replace) {
    // replace 的话 key 还是当前的 key 没必要生成新的
      history.replaceState({ key: _key }, '', url)
    } else {
    // 重新生成 key
      _key = genKey()
       // 带入新的 key 值
      history.pushState({ key: _key }, '', url)
    }
  } catch (e) {
  // 达到限制了 则重新指定新的地址
    window.location[replace ? 'replace' : 'assign'](url)
  }
}
```
#### replaceState

```
// 直接调用 pushState 传入 replace 为 true
export function replaceState (url?: string) {
  pushState(url, true)
}
```
pushState和replaceState两种方法的共同特点：当调用他们修改浏览器历史栈后，虽然当前url改变了，但浏览器不会立即发送请求该url，这就为单页应用前端路由，更新视图但不重新请求页面提供了基础。
#### supportsPushState

```
export const supportsPushState = inBrowser && (function () {
  const ua = window.navigator.userAgent

  if (
    (ua.indexOf('Android 2.') !== -1 || ua.indexOf('Android 4.0') !== -1) &&
    ua.indexOf('Mobile Safari') !== -1 &&
    ua.indexOf('Chrome') === -1 &&
    ua.indexOf('Windows Phone') === -1
  ) {
    return false
  }

  return window.history && 'pushState' in window.history
})()
```
其实所谓响应式属性，即当_route值改变时，会自动调用Vue实例的render()方法，更新视图。
$router.push()-->HashHistory.push()-->History.transitionTo()-->History.updateRoute()-->{app._route=route}-->vm.render()

#### 监听地址栏
在浏览器中，用户可以直接在浏览器地址栏中输入改变路由，因此还需要监听浏览器地址栏中路由的变化 ，并具有与通过代码调用相同的响应行为，在HashHistory中这一功能通过setupListeners监听hashchange实现：

```
setupListeners () {
    window.addEventListener('hashchange', () => {
        if (!ensureSlash()) {
            return
        }
        this.transitionTo(getHash(), route => {
            replaceHash(route.fullPath)
        })
    })
}
```
### HTML5History

History interface是浏览器历史记录栈提供的接口，通过back(),forward(),go()等方法，我们可以读取浏览器历史记录栈的信息，进行各种跳转操作。

```
export class HTML5History extends History {
  constructor (router: Router, base: ?string) {
    super(router, base)

    const expectScroll = router.options.scrollBehavior //指回滚方式
    const supportsScroll = supportsPushState && expectScroll

    if (supportsScroll) {
      setupScroll()
    }

    const initLocation = getLocation(this.base)
    //监控popstate事件
    window.addEventListener('popstate', e => {
      const current = this.current

      // Avoiding first `popstate` event dispatched in some browsers but first
      // history route not updated since async guard at the same time.
      // 避免在某些浏览器中首次发出“popstate”事件
      // 由于同一时间异步监听，history路由没有同时更新。
      const location = getLocation(this.base)
      if (this.current === START && location === initLocation) {
        return
      }

      this.transitionTo(location, route => {
        if (supportsScroll) {
          handleScroll(router, route, current, true)
        }
      })
    })
  }
```
hash模式仅改变hash部分的内容，而hash部分是不会包含在http请求中的(hash带#)：

http://oursite.com/#/user/id //如请求，只会发送http://oursite.com/

所以hash模式下遇到根据url请求页面不会有问题

而history模式则将url修改的就和正常请求后端的url一样(history不带#)

http://oursite.com/user/id

如果这种向后端发送请求的话，后端没有配置对应/user/id的get路由处理,会返回404错误。

官方推荐的解决办法是在服务端增加一个覆盖所有情况的候选资源：如果 URL 匹配不到任何静态资源，则应该返回同一个 index.html 页面，这个页面就是你 app 依赖的页面。同时这么做以后，服务器就不再返回 404 错误页面，因为对于所有路径都会返回 index.html 文件。为了避免这种情况，在 Vue 应用里面覆盖所有的路由情况，然后在给出一个 404 页面。或者，如果是用 Node.js 作后台，可以使用服务端的路由来匹配 URL，当没有匹配到路由的时候返回 404，从而实现 fallback。

#### 两种模式比较
一般的需求场景中，hash模式与history模式是差不多的，根据MDN的介绍，调用history.pushState()相比于直接修改hash主要有以下优势：

• pushState设置的新url可以是与当前url同源的任意url,而hash只可修改#后面的部分，故只可设置与当前同文档的url

• pushState设置的新url可以与当前url一模一样，这样也会把记录添加到栈中，而hash设置的新值必须与原来不一样才会触发记录添加到栈中

• pushState通过stateObject可以添加任意类型的数据记录中，而hash只可添加短字符串
pushState可额外设置title属性供后续使用

### AbstractHistory
'abstract'模式，不涉及和浏览器地址的相关记录，流程跟'HashHistory'是一样的，其原理是通过数组模拟浏览器历史记录栈的功能


```
//abstract.js实现，这里通过栈的数据结构来模拟路由路径
export class AbstractHistory extends History {
  index: number;
  stack: Array<Route>;

  constructor (router: Router, base: ?string) {
    super(router, base)
    this.stack = []
    this.index = -1
  }
  
  // 对于 go 的模拟
  go (n: number) {
    // 新的历史记录位置
    const targetIndex = this.index + n
    // 小于或大于超出则返回
    if (targetIndex < 0 || targetIndex >= this.stack.length) {
      return
    }
    // 取得新的 route 对象
    // 因为是和浏览器无关的 这里得到的一定是已经访问过的
    const route = this.stack[targetIndex]
    // 所以这里直接调用 confirmTransition 了
    // 而不是调用 transitionTo 还要走一遍 match 逻辑
    this.confirmTransition(route, () => {
      this.index = targetIndex
      this.updateRoute(route)
    })
  }
```

```
 //确认是否转化路由
  confirmTransition (route: Route, onComplete: Function, onAbort?: Function) {
    const current = this.current
    const abort = err => {
      if (isError(err)) {
        if (this.errorCbs.length) {
          this.errorCbs.forEach(cb => { cb(err) })
        } else {
          warn(false, 'uncaught error during route navigation:')
          console.error(err)
        }
      }
      onAbort && onAbort(err)
    }
    //判断如果前后是同一个路由，不进行操作
    if (
      isSameRoute(route, current) &&
      route.matched.length === current.matched.length
    ) {
      this.ensureURL()
      return abort()
    }
    //下面是各类钩子函数的处理
    //*********************
    })
  }
```

看到这里你已经对vue-router的路由基本掌握的差不多了，要是喜欢看源码可以[点击查](https://github.com/vuejs/vue-router/tree/dev/src)看

要是喜欢可以给我一个star，[github](https://github.com/DIVIBEAR/vue)

感谢[Aine_潔](https://juejin.im/post/5b08c9ccf265da0dd527d98d)和[CaiBoBo](https://www.cnblogs.com/caizhenbo/p/7297730.html)两位老师提供的思路。









