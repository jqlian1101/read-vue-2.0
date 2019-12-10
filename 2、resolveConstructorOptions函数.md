# resolveConstructorOptions函数  

第一节 入口文件解析中，调用`this._init`函数时，第一步就是对options进行处理，当我们`new Vue`时，对options参数对处理会走`const ctorOption = resolveConstructorOptions(vm.constructor)`，我们看一下resolveConstructorOptions具体对实现，可以看到，这个函数主要的作用是返回Vue构造函数上的options，可以通过调用[Vue.extend](https://cn.vuejs.org/v2/api/#Vue-extend)的可以查看该函数的具体执行  
```javascript
export function resolveConstructorOptions (Ctor: Class<Component>) {
    let options = Ctor.options  // Ctor即Vue$3，
    // Ctor.super 来判断该类是否是Vue的子类
    // 有super属性，说明Ctor是Vue.extend构建的子类
    if (Ctor.super) {
        const superOptions = resolveConstructorOptions(Ctor.super)
        const cachedSuperOptions = Ctor.superOptions    // Vue构造函数上的options,如directives, filters, components ...

        /**
         * 父类中的options 有没有发生变化
         *
         * 当为Vue混入一些options时，superOptions会发生变化，
         * 此时于之前子类中存储的cachedSuperOptions已经不等，
         * 所以下面的操作主要就是更新sub.superOptions
         */
        if (superOptions !== cachedSuperOptions) {
            // super option changed,
            // need to resolve new options.
            Ctor.superOptions = superOptions
            // check if there are any late-modified/attached options (#4976)
            const modifiedOptions = resolveModifiedOptions(Ctor)
            // update base extend options
            if (modifiedOptions) {
                extend(Ctor.extendOptions, modifiedOptions)
            }

            // 调用mergeOptions合并父类的options(superOptions)和Ctor的extendOptions
            options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions)

            if (options.name) {
                options.components[options.name] = Ctor
            }
        }
    }

    return options
}
```  

可以看到，如果Ctor上有super属性，就会执行if代码块里的内容，super属性可参阅Vue.extend里的内容  
```javascript
 Vue.extend = function (extendOptions: Object): Function {
    ...
    Sub['super'] = Super
    ...
}
```  

if代码块中，首先会递归调用resolveConstructorOptions方法，返回"父类"上的options并赋值给superOptions变量。然后把"自身"的options赋值给cachedSuperOptions变量。  
然后比较这两个变量的值,当这两个变量值不等时，说明"父类"的options改变过了。例如执行了Vue.mixin方法，这时候就需要把"自身"的superOptions属性替换成最新的。然后通过resolveModifiedOptions检查"自身"的options是否发生变化  
举例说明：  
```javascript
var Profile = Vue.extend({
    template: '<p>{{firstName}} {{lastName}} aka {{alias}}</p>'
})

Vue.mixin({
    data: function () {
        return {
            firstName: 'Walter',
            lastName: 'White',
            alias: 'Heisenberg'
        }
    }
})
new Profile().$mount('#example')
```  

由于[Vue.mixin](https://cn.vuejs.org/v2/guide/mixins.html#%E5%85%A8%E5%B1%80%E6%B7%B7%E5%85%A5)改变了"父类"options，此时superOptions和cachedSuperOptions就不相等了  
```javascript
function resolveModifiedOptions (Ctor: Class<Component>): ?Object {
    let modified
    const latest = Ctor.options         // 自身的options
    const extended = Ctor.extendOptions // Vue.extend传入的options，详见Vue.extend
    const sealed = Ctor.sealedOptions   // 执行Vue.extend时封装的"自身"options，这个属性就是方便检查"自身"的options有没有变化，详见Vue.extend

    // 遍历当前构造器上的options属性，如果在"自身"封装的options里没有，则证明是新添加的。执行if内的语句。调用dedupe方法，最终返回modified变量(即”自身新添加的options“)
    for (const key in latest) {
        if (latest[key] !== sealed[key]) {
            if (!modified) modified = {}
            modified[key] = dedupe(latest[key], extended[key], sealed[key])
        }
    }
    return modified
}
```  

最后，调用mergeOptions合并父类的options(superOptions)和Ctor的extendOptions，同时，执行完resolveConstructorOptions后，_init函数会调用mergeOptions函数对options进行合并，后续将对mergeOptions进行分析


