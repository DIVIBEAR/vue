这一章主要讲的是在render的时候如何做到对data中元素进行依赖的收集与监听。

下面让我们看看这行代码：

```
new Vue({
    template: 
        `<div>
            <span>span1:</span> {{span1}}
            <span>span2:</span> {{span2}}
        <div>`,
    data: {
        span1: 'span1',
        span2: 'span2',
        span3: 'span3'
    }
});
```
按照之前我们理解的data中依赖收集与监听方法进行绑定则会出现一个问题——span3在实际模板中并没有被用到，然而当span3的数据被修改的时候（this.span3 = 'span4'）的时候，同样会触发span3的setter导致重新执行渲染，这显然不符合我们所想要的。

## Dep
当对data上的对象进行修改值的时候会触发它的setter，那么取值的时候自然就会触发getter事件，所以我们只要在最开始初始化的时候进行一次render，那么**所有被渲染所依赖的data中的数据**就会被getter收集到Dep的subs中去。在对data中的数据进行修改的时候setter只会触发Dep的subs的函数。

定义一个依赖收集类Dep。


```
class Dep {
    constructor () {
        this.subs = [];
    }
    /**进行依赖的收集**/
    addSub (sub: Watcher) {
        this.subs.push(sub)
    }
    /**进行依赖的移除**/
    removeSub (sub: Watcher) {
        remove(this.subs, sub)
    }
    /*Github:https://github.com/answershuto*/
    /**进行依赖的遍历通知**/
    notify () {
        // stabilize the subscriber list first
        const subs = this.subs.slice()
        for (let i = 0, l = subs.length; i < l; i++) {
            subs[i].update()
        }
    }
}
function remove (arr, item) {
    if (arr.length) {
        const index = arr.indexOf(item)
        if (index > -1) {
            return arr.splice(index, 1)
    }
}
```

## Watcher

订阅者，当依赖收集的时候会addSub到sub中，在修改data中数据的时候会触发dep对象的notify，通知所有Watcher对象去修改对应视图。

```
class Watcher {
    constructor (vm, expOrFn, cb, options) {
        this.cb = cb;
        this.vm = vm;

        /*在这里将观察者本身赋值给全局的target，只有被target标记过的才会进行依赖收集*/
        Dep.target = this;
        /*Github:https://github.com/answershuto*/
        /*触发渲染操作进行依赖收集*/
        this.cb.call(this.vm);
    }

    update () {
        this.cb.call(this.vm);
    }
}
```
## 开始依赖收集

```
class Vue {
    constructor(options) {
        this._data = options.data;
        observer(this._data, options.render);
        let watcher = new Watcher(this, );
    }
}

function defineReactive (obj, key, val, cb) {
    /*在闭包内存储一个Dep对象*/
    const dep = new Dep();

    Object.defineProperty(obj, key, {
        enumerable: true,
        configurable: true,
        get: ()=>{
            if (Dep.target) {
                /*Watcher对象存在全局的Dep.target中*/
                dep.addSub(Dep.target);
            }
        },
        set:newVal=> {
            /*只有之前addSub中的函数才会触发*/
            dep.notify();
        }
    })
}

Dep.target = null;
```
将观察者Watcher实例赋值给全局的Dep.target，然后触发render操作只有被Dep.target标记过的才会进行依赖收集。有Dep.target的对象会将Watcher的实例push到subs中，在对象被修改出发setter操作的时候dep会调用subs中的Watcher实例的update方法进行渲染。

详细过程可以参考我之前写的[浅析Vue源码（三）—— initMixin(下)](https://juejin.im/post/5bb48c43e51d450e8b13e4ac)，在这里面具体介绍了Dep,Watcher,defineReactive的方法。

感谢[染陌老师](https://github.com/answershuto/learnVue)提供的思路。
要是喜欢的话可以给我个star, [github](https://github.com/DIVIBEAR/vue)
