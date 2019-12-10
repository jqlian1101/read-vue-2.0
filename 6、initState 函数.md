# initState 函数  
先来看一下 `initState`函数主要做了什么   
`src/core/instance/state.js`   
```javascript
export function initState (vm: Component) {
    vm._watchers = []
    const opts = vm.$options

    if (opts.props) initProps(vm, opts.props)

    if (opts.methods) initMethods(vm, opts.methods)

    if (opts.data) {
        initData(vm)
    } else {
        observe(vm._data = {}, true /* asRootData */)
    }

    if (opts.computed) initComputed(vm, opts.computed)

    if (opts.watch && opts.watch !== nativeWatch) {
        initWatch(vm, opts.watch)
    }
}
```  

可以看到，`initState`内部很简单，主要是vm状态的初始化，`props/methods/data/computed//watch`都在这里完成初始化，因此该函数也是Vue实例create的关键。下面简单看一下各属性初始化的过程。  

#### initProps

`initProps`主要对`props`进行校验，并通过`defineReactive`进行可监听处理
```javascript
function initProps (vm: Component, propsOptions: Object) {
    const propsData = vm.$options.propsData || {}
    const props = vm._props = {}
    // cache prop keys so that future props updates can iterate using Array
    // instead of dynamic object key enumeration.
    // 用于保存当前组件的props里的key; 以便之后在父组件更新props时可以直接使用数组迭代，而不需要动态枚举键值
    const keys = vm.$options._propKeys = []
    const isRoot = !vm.$parent

    // root instance props should be converted
    observerState.shouldConvert = isRoot

    for (const key in propsOptions) {
        keys.push(key)

        // 执行validateProp检查propsData里的key值是否符合propsOptions里对应的要求，并将值保存到value里面
        const value = validateProp(key, propsOptions, propsData, vm)

        ... 

        // 进行可监听处理
        defineReactive(props, key, value)
        
        ...

        if (!(key in vm)) {
            proxy(vm, `_props`, key)
        }
    }

    observerState.shouldConvert = true
}
```
#### initMethods
`initMethods`的作用就是将options.methods里定义的方法挂载到vm上  
```javascript
function initMethods (vm: Component, methods: Object) {
    const props = vm.$options.props
    for (const key in methods) {
        ...
        vm[key] = methods[key] == null ? noop : bind(methods[key], vm)
    }
}
```

#### initData
`initData`中，通过vm.xx来代理data.xx，再通过`observe`函数将data设置为可监听
```javascript
function initData (vm: Component) {
    let data = vm.$options.data
    data = vm._data = typeof data === 'function'
        ? getData(data, vm)
        : data || {}

    if (!isPlainObject(data)) {
        data = {}
        ...
    }
    // proxy data on instance
    const keys = Object.keys(data)
    const props = vm.$options.props
    const methods = vm.$options.methods

    let i = keys.length
    while (i--) {
        const key = keys[i]
        ...
        // 代理数据，当访问```this.xxx```的时候，代理到```this.data.xxx```
        proxy(vm, `_data`, key)
    }

    // observe data
    observe(data, true /* asRootData */)
}
```

#### initComputed
`initComputed`为每个computed创建watcher
```javascript
function initComputed (vm: Component, computed: Object) {
    const watchers = vm._computedWatchers = Object.create(null)
    const isSSR = isServerRendering()

    for (const key in computed) {
        const userDef = computed[key]
        const getter = typeof userDef === 'function' ? userDef : userDef.get
        
        ...

        // 为computed创建watcher
        watchers[key] = new Watcher(
            vm,
            getter || noop,
            noop,
            computedWatcherOptions
        )
        
        ...

        defineComputed(vm, key, userDef)
        
        ...
    }
}
```

`defineComputed`通过`defineProperty`将computed挂载到vm上
```javascript
export function defineComputed (
    target: any,
    key: string,
    userDef: Object | Function
) {
    // 如果非服务端渲染，进行缓存处理
    const shouldCache = !isServerRendering()

    // 定义setter和getter
    if (typeof userDef === 'function') {
        sharedPropertyDefinition.get = shouldCache
            ? createComputedGetter(key)
            : userDef
        sharedPropertyDefinition.set = noop
    } else {
        sharedPropertyDefinition.get = userDef.get
            ? shouldCache && userDef.cache !== false
                ? createComputedGetter(key)
                : userDef.get
            : noop
        sharedPropertyDefinition.set = userDef.set
            ? userDef.set
            : noop
    }
    
    ...
    
    // 将computed挂载到vm实例上
    Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

`createComputedGetter`
```javascript
function createComputedGetter (key) {
    return function computedGetter () {
        const watcher = this._computedWatchers && this._computedWatchers[key]
        if (watcher) {
            // 是否需要重新计算watcher.value
            if (watcher.dirty) {
                watcher.evaluate()
            }

            // 收集依赖
            if (Dep.target) {
                watcher.depend()
            }

            return watcher.value
        }
    }
}
```

#### initWatch
`initWatch`通过option.watch的不同类型进行处理，最后调用`createWatcher`来构建`Watcher`对象
```javascript
function initWatch (vm: Component, watch: Object) {
    for (const key in watch) {
        const handler = watch[key]
        if (Array.isArray(handler)) {
            for (let i = 0; i < handler.length; i++) {
                createWatcher(vm, key, handler[i])
            }
        } else {
            createWatcher(vm, key, handler)
        }
    }
}
```

`createWatcher`  
```javascript
function createWatcher (
    vm: Component,
    keyOrFn: string | Function,
    handler: any,
    options?: Object
) {
    // handler 是obj时，必须设置handler.handler，
    // obj包含 handler，deep，immediate，
    // eg: https://cn.vuejs.org/v2/api/#watch
    if (isPlainObject(handler)) {
        options = handler
        handler = handler.handler
    }

    // 回调函数是一个字符串，从 vm 获取
    if (typeof handler === 'string') {
        handler = vm[handler]
    }

    // expOrFn 是 key，options 是 watch 的全部选项
    return vm.$watch(keyOrFn, handler, options)
}
```

`$watch`
```javascript
Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
): Function {
    const vm: Component = this
    if (isPlainObject(cb)) {
        return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {}
    options.user = true

    // expOrFn是监听的key，cb是监听的回调，options是监听的所有选项
    const watcher = new Watcher(vm, expOrFn, cb, options)

    // 如果设置immediate: true 将立即以表达式的当前值触发回调
    // https://cn.vuejs.org/v2/api/#vm-watch
    if (options.immediate) {
        cb.call(vm, watcher.value)
    }

    // 返回unWatch的方法
    return function unwatchFn () {
        watcher.teardown()
    }
}
```  
