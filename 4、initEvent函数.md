# initEvent函数

接着前面的章节，_init函数中，执行完`mergeOptions`以后，接着执行一下函数
```javascript
initLifecycle(vm)   // vm的生命周期相关变量初始化
initEvents(vm)  // vm的事件监听初始化
initRender(vm)  // 初始化渲染函数
callHook(vm, 'beforeCreate')    // 回调 beforeCreate 钩子函数
initInjections(vm) // resolve injections before data/props

// vm的状态初始化，prop/data/computed/method/watch都在这里完成初始化，因此也是Vue实例create的关键。
initState(vm)
initProvide(vm) // resolve provide after data/props
callHook(vm, 'created') // vm 已经创建好 回调 created 钩子函数
```  

// TODO initLifecyle源码连接
其中，initLifecycle比较简单，可自行翻看源码注释，本节主要讲解initEvent函数

`src/core/instance/events.js`  
```javascript
export function initEvents (vm: Component) {
    // 创建事件对象，用于存储事件
    vm._events = Object.create(null)

    vm._hasHookEvent = false

    // init parent attached events
    // _parentListeners其实是父组件模板中写的v-on
    // 所以下面这段就是将父组件模板中注册的事件放到当前组件实例的listeners里面
    const listeners = vm.$options._parentListeners
    if (listeners) {
        updateComponentListeners(vm, listeners)
    }
}
```  

其中，_hasHookEvent用于标识组件是否有通过`@hook:`绑定的事件，例如：

```html
<script type='text/x-template' id='demo'>
    <div>
        <template1
            :message="message"
            @hook:created="createdHookFn"
            @onMouseover="onMouseover"
        />
    </div>
</script>

<script type='text/x-template' id='template1'>
    <div @click="onClickTemp" @mouseover="$emit('onMouseover')">{{ message }}</div>
</script>
```
```javascript
Vue.component('template1', {
    template: '#template1',
    data: () => {
        return {}
    },
    props: ['message'],
    created() {
        console.log('template1 created')
    },
    methods: {
        onClickTemp() {
            console.log('onClickTemp')
        }
    }
})

new Vue({
    el: '#app',
    template: '#demo',
    components: {
    },
    data: () => {
        return {
            message: '这是一条消息',
        }
    },
    methods: {
        createdHookFn() {
            console.log('createdHookFn')
        },
        onMouseover() {
            console.log('onMouseover')
        }
    }
})
```  

例子中，child组件被调用的时候，父组件传递的两个属性：`hook:created`和`onMouseover`，将会被存储在child实例的_events中；当调用initEvent当时候，会调用`updateComponentListeners`来处理父组件传递当事件；updateComponentListeners中很简单，只是调用了updateListeners，下面看一下updateListeners：  

```javascript
export function updateListeners (
    on: Object,
    oldOn: Object,
    add: Function,
    remove: Function,
    vm: Component
) {
    let name, cur, old, event

    for (name in on) {
        cur = on[name]
        old = oldOn[name]

        // 处理once (～)，capture (!)，passive (&) 事件修饰符
        event = normalizeEvent(name)

        if (isUndef(cur)) {
            process.env.NODE_ENV !== 'production' && warn(
                `Invalid handler for event "${event.name}": got ` + String(cur),
                vm
            )
        } else if (isUndef(old)) {
            if (isUndef(cur.fns)) {
                cur = on[name] = createFnInvoker(cur)
            }
            // 添加事件监听
            add(event.name, cur, event.once, event.capture, event.passive)
        } else if (cur !== old) {
            old.fns = cur
            on[name] = old
        }
    }

    // 移除旧的事件监听
    for (name in oldOn) {
        if (isUndef(on[name])) {
            event = normalizeEvent(name)
            remove(event.name, oldOn[name], event.capture)
        }
    }
}

```
updateListeners中，主要是调用了add函数，来添加监听事件
```javascript
function add (event, fn, once) {
    if (once) {
        target.$once(event, fn)
    } else {
        target.$on(event, fn)
    }
}
```
可以看到，add中，直接调用了$on，$on中，主要将event存储到vm._events中，并判断是否是通过hook:event来进行绑定的来设置_hasHookEvent的值

`src/core/instance/events.js`  

```javascript
export function eventsMixin (Vue: Class<Component>) {
    const hookRE = /^hook:/

    Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
        const vm: Component = this
        if (Array.isArray(event)) {
            for (let i = 0, l = event.length; i < l; i++) {
                this.$on(event[i], fn)
            }
        } else {
            // 缓存events处理函数
            (vm._events[event] || (vm._events[event] = [])).push(fn)

            // 判断是否是通过hook:event来进行绑定的
            // 如果是，在执行callHook的时候，会$emit相应的钩子函数
            if (hookRE.test(event)) {
                vm._hasHookEvent = true
            }
        }
        return vm
    }
    ...
}
```  

_hasHookEvent的作用就是当调用个钩子函数的时候，判断是否需要调用$emit来触发绑定的hookEvent函数

```javascript
export function callHook (vm: Component, hook: string) {
    ...
    if (vm._hasHookEvent) {
        vm.$emit('hook:' + hook)
    }
    ...
}
```  
_events中的其他事件，将在子组件调用$emit的时候，进行调用。  


