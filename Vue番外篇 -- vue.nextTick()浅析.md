当我们在vue的beforeCreate和created生命周期发送ajax到后台，数据返回的时候发现DOM节点还未生成无法操作节点，那要怎么办呢？

这时，我们就会用到一个方法是this.$nextTick（相信你也用过）。

nextTick是全局vue的一个函数，在vue系统中，用于处理dom更新的操作。vue里面有一个watcher，用于观察数据的变化，然后更新dom，vue里面并不是每次数据改变都会触发更新dom，而是将这些操作都缓存在一个队列，在一个事件循环结束之后，刷新队列，统一执行dom更新操作。 

通常情况下，我们不需要关心这个问题，而如果想在DOM状态更新后做点什么，则需要用到nextTick。在vue生命周期的created()钩子函数进行的DOM操作要放在Vue.nextTick()的回调函数中，因为created()钩子函数执行的时候DOM并未进行任何渲染，而此时进行DOM操作是徒劳的，所以此处一定要将DOM操作的JS代码放进Vue.nextTick()的回调函数中。而与之对应的mounted钩子函数，该钩子函数执行时所有的DOM挂载和渲染都已完成，此时该钩子函数进行任何DOM操作都不会有个问题。 

`Vue.nextTick(callback)`，当数据发生变化，更新后执行回调。

`Vue.$nextTick(callback)`，当dom发生变化，更新后执行的回调。

请看如下一段代码:

```
<template>
  <div>
    <div ref="text">{{text}}</div>
    <button @click="handleClick">text</button>
  </div>
</template>
export default {
    data () {
        return {
            text: 'start'
        };
    },
    methods () {
        handleClick () {
            this.text = 'end';
            console.log(this.$refs.text.innerText);//打印“start”
        }
    }
}
```
打印的结果是start，为什么明明已经将text设置成了“end”，获取真实DOM节点的innerText却没有得到我们预期中的“end”，而是得到之前的值“start”呢？

## 源码解读
带着这个疑问，我们找到了Vue.js源码的Watch实现。当某个响应式数据发生变化的时候，它的setter函数会通知闭包中的Dep，Dep则会调用它管理的所有Watch对象。触发Watch对象的update实现。我们来看一下update的实现。

### watcher

```
/*
      调度者接口，当依赖发生改变的时候进行回调。
   */
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
    /*同步则执行run直接渲染视图*/
      this.run()
    } else {
    /*异步推送到观察者队列中，由调度者调用。*/
      queueWatcher(this)
    }
  }
```
我们发现Vue.js默认是使用异步执行DOM更新。
当异步执行update的时候，会调用queueWatcher函数。

```
/**
 * Push a watcher into the watcher queue.
 * Jobs with duplicate IDs will be skipped unless it's
 * pushed when the queue is being flushed.
 **/
 /*将一个观察者对象push进观察者队列，在队列中已经存在相同的id则该观察者对象将被跳过，除非它是在队列被刷新时推送*/
export function queueWatcher (watcher: Watcher) {
    /*获取watcher的id*/
  const id = watcher.id
   /*检验id是否存在，已经存在则直接跳过，不存在则标记哈希表has，用于下次检验*/
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
    /*如果没有flush掉，直接push到队列中即可*/
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    // 刷新队列
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```
查看queueWatcher的源码我们发现，Watch对象并不是立即更新视图，而是被push进了一个队列queue，此时状态处于waiting的状态，这时候继续会有Watch对象被push进这个队列queue，等待下一个tick时，这些Watch对象才会被遍历取出，更新视图。同时，id重复的Watcher不会被多次加入到queue中去，因为在最终渲染时，我们只需要关心数据的最终结果。

### flushSchedulerQueue

```
vue/src/core/observer/scheduler.js
```

```
/**
 * Flush both queues and run the watchers.
 */
  /*nextTick的回调函数，在下一个tick时flush掉两个队列同时运行watchers*/
function flushSchedulerQueue () {
  flushing = true
  let watcher, id

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  /*
    刷新前给queue排序，这样做可以保证：
    1.组件更新的顺序是从父组件到子组件的顺序，因为父组件总是比子组件先创建。
    2.一个组件的user watchers比render watcher先运行，因为user watchers往往比render watcher更早创建
    3.如果一个组件在父组件watcher运行期间被销毁，它的watcher执行将被跳过。
  */
  queue.sort((a, b) => a.id - b.id)

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  /*这里不用index = queue.length;index > 0; index--的方式写是因为不要将length进行缓存，
  因为在执行处理现有watcher对象期间，更多的watcher对象可能会被push进queue*/
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
     /*将has的标记删除*/
    has[id] = null
     /*执行watcher*/
    watcher.run()
    // in dev build, check and stop circular updates.
    /*
      在测试环境中，检测watch是否在死循环中
      比如这样一种情况
      watch: {
        test () {
          this.test++;
        }
      }
      持续执行了一百次watch代表可能存在死循环
    */
    if (process.env.NODE_ENV !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? `in watcher with expression "${watcher.expression}"`
              : `in a component render function.`
          ),
          watcher.vm
        )
        break
      }
    }
  }
  // keep copies of post queues before resetting state
  /*得到队列的拷贝*/
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()
  /*重置调度者的状态*/
  resetSchedulerState()

  // call component updated and activated hooks
  /*使子组件状态都改编成active同时调用activated钩子*/
  callActivatedHooks(activatedQueue)
  /*调用updated钩子*/
  callUpdatedHooks(updatedQueue)

  // devtool hook
  /* istanbul ignore if */
  if (devtools && config.devtools) {
    devtools.emit('flush')
  }
}
```
flushSchedulerQueue是下一个tick时的回调函数，主要目的是执行Watcher的run函数，用来更新视图


### nextTick
vue.js提供了一个nextTick函数，其实也就是上面调用的nextTick。

nextTick的实现比较简单，执行的目的是在microtask或者task中推入一个funtion，在当前栈执行完毕（也行还会有一些排在前面的需要执行的任务）以后执行nextTick传入的funtion。

网上很多文章讨论的nextTick实现是2.4版本以下的实现，2.5以上版本对于nextTick的内部实现进行了大量的修改，看一下源码：

首先是从Vue 2.5+开始，抽出来了一个单独的文件next-tick.js来执行它。
```
vue/src/core/util/next-tick.js
```

```
 /*
    延迟一个任务使其异步执行，在下一个tick时执行，一个立即执行函数，返回一个function
    这个函数的作用是在task或者microtask中推入一个timerFunc，
    在当前调用栈执行完以后以此执行直到执行到timerFunc
    目的是延迟到当前调用栈执行完以后执行
*/
/*存放异步执行的回调*/
const callbacks = []
/*一个标记位，如果已经有timerFunc被推送到任务队列中去则不需要重复推送*/
let pending = false

/*下一个tick时的回调*/
function flushCallbacks () {
/*一个标记位，标记等待状态（即函数已经被推入任务队列或者主线程，已经在等待当前栈执行完毕去执行），这样就不需要在push多个回调到callbacks时将timerFunc多次推入任务队列或者主线程*/
  pending = false
  //复制callback
  const copies = callbacks.slice(0)
  //清除callbacks
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
  //触发callback的回调函数
    copies[i]()
  }
}

// Here we have async deferring wrappers using both microtasks and (macro) tasks.
// In < 2.4 we used microtasks everywhere, but there are some scenarios where
// microtasks have too high a priority and fire in between supposedly
// sequential events (e.g. #4521, #6690) or even between bubbling of the same
// event (#6566). However, using (macro) tasks everywhere also has subtle problems
// when state is changed right before repaint (e.g. #6813, out-in transitions).
// Here we use microtask by default, but expose a way to force (macro) task when
// needed (e.g. in event handlers attached by v-on).
/**
其大概的意思就是：在Vue2.4之前的版本中，nextTick几乎都是基于microTask实现的，
但是由于microTask的执行优先级非常高，在某些场景之下它甚至要比事件冒泡还要快，
就会导致一些诡异的问题；但是如果全部都改成macroTask，对一些有重绘和动画的场
景也会有性能的影响。所以最终nextTick采取的策略是默认走microTask，对于一些DOM
的交互事件，如v-on绑定的事件回调处理函数的处理，会强制走macroTask。
**/

let microTimerFunc
let macroTimerFunc
let useMacroTask = false

// Determine (macro) task defer implementation.
// Technically setImmediate should be the ideal choice, but it's only available
// in IE. The only polyfill that consistently queues the callback after all DOM
// events triggered in the same loop is by using MessageChannel.
/* istanbul ignore if */
/**
而对于macroTask的执行，Vue优先检测是否支持原生setImmediate（高版本IE和Edge支持），
不支持的话再去检测是否支持原生MessageChannel，如果还不支持的话为setTimeout(fn, 0)。
**/

if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  macroTimerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else if (typeof MessageChannel !== 'undefined' && ( 
// MessageChannel与原先的MutationObserver异曲同工
/**
在Vue 2.4版本以前使用的MutationObserver来模拟异步任务。
而Vue 2.5版本以后，由于兼容性弃用了MutationObserver。
Vue 2.5+版本使用了MessageChannel来模拟macroTask。
除了IE以外，messageChannel的兼容性还是比较可观的。
**/
  isNative(MessageChannel) ||
  // PhantomJS
  MessageChannel.toString() === '[object MessageChannelConstructor]'
)) {
  /**
  可见，新建一个MessageChannel对象，该对象通过port1来检测信息，port2发送信息。
  通过port2的主动postMessage来触发port1的onmessage事件，
  进而把回调函数flushCallbacks作为macroTask参与事件循环。
  **/
  const channel = new MessageChannel()
  const port = channel.port2
  channel.port1.onmessage = flushCallbacks
  macroTimerFunc = () => {
    port.postMessage(1)
  }
} else {
  /* istanbul ignore next */
   //上面两种都不支持，用setTimeout
  macroTimerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

// Determine microtask defer implementation.
/* istanbul ignore next, $flow-disable-line */

if (typeof Promise !== 'undefined' && isNative(Promise)) {
/*使用Promise*/
  const p = Promise.resolve()
  microTimerFunc = () => {
    p.then(flushCallbacks)
    // in problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    //iOS的webview下，需要强制刷新队列，执行上面的回调函数
    if (isIOS) setTimeout(noop)
  }
} else {
  // fallback to macro
  microTimerFunc = macroTimerFunc
}

/**
 * Wrap a function so that if any code inside triggers state change,
 * the changes are queued using a (macro) task instead of a microtask.
 */
 /**
 在Vue执行绑定的DOM事件时，默认会给回调的handler函数调用withMacroTask方法做一层包装，
 它保证整个回调函数的执行过程中，遇到数据状态的改变，这些改变而导致的视图更新（DOM更新）
 的任务都会被推到macroTask而不是microtask。
 **/
export function withMacroTask (fn: Function): Function {
  return fn._withTask || (fn._withTask = function () {
    useMacroTask = true
    const res = fn.apply(null, arguments)
    useMacroTask = false
    return res
  })
}
 /*
    推送到队列中下一个tick时执行
    cb 回调函数
    ctx 上下文
  */
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
   /*cb存到callbacks中*/
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  
  if (!pending) {
    pending = true
    if (useMacroTask) {
      macroTimerFunc()
    } else {
      microTimerFunc()
    }
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}

```
#### MessageChannel VS setTimeout 
为什么要优先MessageChannel创建macroTask而不是setTimeout？

HTML5中规定setTimeout的最小时间延迟是4ms，也就是说理想环境下异步回调最快也是4ms才能触发。

Vue使用这么多函数来模拟异步任务，其目的只有一个，就是让回调异步且尽早调用。而MessageChannel的延迟明显是小于setTimeout的。

说了这么多，到底什么是macrotasks，什么是microtasks呢？

##### 两者的具体实现

**macrotasks：**

setTimeout ，setInterval， setImmediate，requestAnimationFrame, I/O ，UI渲染

**microtasks:**

Promise， process.nextTick， Object.observe， MutationObserver

再简单点可以总结为：

![](https://user-gold-cdn.xitu.io/2018/10/20/1668d443b5646254?w=720&h=343&f=jpeg&s=13315)
**1.在 macrotask 队列中执行最早的那个 task ，然后移出**

**2.再执行 microtask 队列中所有可用的任务，然后移出**

**3.下一个循环，执行下一个 macrotask 中的任务 (再跳到第2步)**

那我们上面提到的任务队列到底是什么呢？跟macrotasks和microtasks有什么联系呢？

```
• An event loop has one or more task queues.(task queue is macrotask queue)
• Each event loop has a microtask queue.
• task queue = macrotask queue != microtask queue
• a task may be pushed into macrotask queue,or microtask queue
• when a task is pushed into a queue(micro/macro),we mean preparing work is finished,
so the task can be executed now.
```
翻译一下就是：

• 一个事件循环有一个或者多个任务队列；

• 每个事件循环都有一个microtask队列；

• macrotask队列就是我们常说的任务队列，microtask队列不是任务队列；

• 一个任务可以被放入到macrotask队列，也可以放入microtask队列；

• 当一个任务被放入microtask或者macrotask队列后，准备工作就已经结束，这时候可以开始执行任务了。

可见，setTimeout和Promises不是同一类的任务，处理方式应该会有区别，具体的处理方式有什么不同呢？从[这篇文章](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/?utm_source=html5weekly)里找到了下面这段话：

```
Microtasks are usually scheduled for things that should happen straight after the currently
executing script, such as reacting to a batch of actions, or to make something async
without taking the penalty of a whole new task. The microtask queue is processed after
callbacks as long as no other JavaScript is mid-execution, and at the end of each task. Any
additional microtasks queued during microtasks are added to the end of the queue and also
processed. Microtasks include mutation observer callbacks, and as in the above example,
promise callbacks.
```
通俗的解释一下，microtasks的作用是用来调度应在当前执行的脚本执行结束后立即执行的任务。 例如响应事件、或者异步操作，以避免付出额外的一个task的费用。

microtask会在两种情况下执行：

任务队列(macrotask = task queue)回调后执行，前提条件是当前没有其他执行中的代码。
每个task末尾执行。
另外在处理microtask期间，如果有新添加的microtasks，也会被添加到队列的末尾并执行。

也就是说执行顺序是：

开始 -> 取task queue第一个task执行 -> 取microtask全部任务依次执行 -> 取task queue下一个任务执行 -> 再次取出microtask全部任务执行 -> ... 这样循环往复

```
Once a promise settles, or if it has already settled, it queues a microtask for its
reactionary callbacks. This ensures promise callbacks are async even if the promise has
already settled. So calling .then(yey, nay) against a settled promise immediately queues a
microtask. This is why promise1 and promise2 are logged after script end, as the currently
running script must finish before microtasks are handled. promise1 and promise2 are logged
before setTimeout, as microtasks always happen before the next task.
```
Promise一旦状态置为完成态，便为其回调(.then内的函数)安排一个microtask。

接下来我们看回我们上面的代码：

```
setTimeout(function(){
    console.log(1)
},0);
new Promise(function(resolve){
    console.log(2)
    for( var i=100000 ; i>0 ; i-- ){
        i==1 && resolve()
    }
    console.log(3)
}).then(function(){
    console.log(4)
});
console.log(5);
```
按照上面的规则重新分析一遍：

当运行到setTimeout时，会把setTimeout的回调函数console.log(1)放到任务队列里去，然后继续向下执行。

接下来会遇到一个Promise。首先执行打印console.log(2)，然后执行for循环，即时for循环要累加到10万，也是在执行栈里面，等待for循环执行完毕以后，将Promise的状态从fulfilled切换到resolve，随后把要执行的回调函数，也就是then里面的console.log(4)推到microtask里面去。接下来马上执行马上console.log(3)。

然后出Promise，还剩一个同步的console.log(5)，直接打印。这样第一轮下来，已经依次打印了2，3，5。

现在第一轮任务队列已经执行完毕，没有正在执行的代码。符合上面讲的microtask执行条件，因此会将microtask中的任务优先执行，因此执行console.log(4)

最后还剩macrotask里的setTimeout放入的函数console.log(1)最后执行。

如此分析输出顺序是：

```
2
3
5
4
1
```
我们再来看一个：

当一个程序有：setTimeout， setInterval ，setImmediate， I/O， UI渲染，Promise ，process.nextTick， Object.observe， MutationObserver的时候：

1.先执行 macrotasks：I/O -》 UI渲染

2.再执行 microtasks ：process.nextTick -》 Promise -》MutationObserver ->Object.observe

3.再把setTimeout setInterval setImmediate 塞入一个新的macrotasks，依次：

setTimeout ，setInterval --》setImmediate

综上，nextTick的目的就是产生一个回调函数加入task或者microtask中，当前栈执行完以后（可能中间还有别的排在前面的函数）调用该回调函数，起到了异步触发（即下一个tick时触发）的目的。

```
setImmediate(function(){
    console.log(1);
},0);
setTimeout(function(){
    console.log(2);
},0);
new Promise(function(resolve){
    console.log(3);
    resolve();
    console.log(4);
}).then(function(){
    console.log(5);
});
console.log(6);
process.nextTick(function(){
    console.log(7);
});
console.log(8);
结果是：3 4 6 8 7 5 2 1
```
### 使用了nextTick异步更新视图有什么好处呢？
接下来我们看一下一个Demo：

```
<template>
  <div>
    <div>{{test}}</div>
  </div>
</template>
export default {
    data () {
        return {
            test: 0
        };
    },
    created () {
      for(let i = 0; i < 1000; i++) {
        this.test++;
      }
    }
}
```
现在有这样的一种情况，created的时候test的值会被++循环执行1000次。
每次++时，都会根据响应式触发setter->Dep->Watcher->update->patch。
如果这时候没有异步更新视图，那么每次++都会直接操作DOM更新视图，这是非常消耗性能的。
所以Vue.js实现了一个queue队列，在下一个tick的时候会统一执行queue中Watcher的run。同时，拥有相同id的Watcher不会被重复加入到该queue中去，所以不会执行1000次Watcher的run。最终更新视图只会直接将test对应的DOM的0变成1000。
保证更新视图操作DOM的动作是在当前栈执行完以后下一个tick的时候调用，大大优化了性能。

要是喜欢的话给个star, 鼓励一下，[github](https://github.com/DIVIBEAR/vue)

感谢[首发于](https://zhuanlan.zhihu.com/p/30451651)和[zhleven](https://www.jianshu.com/p/d3ee32538b53)提供的思路。





