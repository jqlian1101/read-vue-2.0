# $mount  

第一节说过，vue源码入口`src/platform/web/entry-runtime-with-compiler.js`，在这个文件中，重写了`Vue.prototype.$mount`，添加了模版编译功能。  

```javascript
// 把原本不带编译的$mount方法保存下来，在最后会调用
const mount = Vue.prototype.$mount

// 挂载组件，带模板编译
Vue.prototype.$mount = function (
    el?: string | Element,
    hydrating?: boolean
): Component {
    // 通过query方法重写el(挂载点: 组件挂载的占位符)
    el = el && query(el)

    /* istanbul ignore if */
    // 提示不能把body/html作为挂载点, 开发环境下给出错误提示
    // 因为挂载点是会被组件模板自身替换点, 显然body/html不能被替换
    if (el === document.body || el === document.documentElement) {
        process.env.NODE_ENV !== 'production' && warn(
            `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
        )
        return this
    }

    // $options是在new Vue(options)时候_init方法内执行.
    // $options可以访问到options的所有属性如data, filter, components, directives等
    const options = this.$options

    // resolve template/el and convert to render function
    // 处理模板template，编译成render函数，render不存在的时候才会编译template，否则优先使用render
    if (!options.render) {
        // 没有render函数时候优先考虑template属性
        let template = options.template
        if (template) {
            // template存在且template的类型是字符串
            if (typeof template === 'string') {
                if (template.charAt(0) === '#') {
                    // template是ID
                    template = idToTemplate(template)
                    
                    /* istanbul ignore if */
                    if (process.env.NODE_ENV !== 'production' && !template) {
                        warn(
                            `Template element not found or is empty: ${options.template}`,
                            this
                        )
                    }
                }
            } else if (template.nodeType) {
                // template 的类型是元素节点,则使用该元素的 innerHTML 作为模板
                template = template.innerHTML
            } else {
                // 若 template既不是字符串又不是元素节点，那么在开发环境会提示开发者传递的 template 选项无效
                if (process.env.NODE_ENV !== 'production') {
                    warn('invalid template option:' + template, this)
                }
                return this
            }
        } else if (el) {
            // 如果template选项不存在，那么使用el元素的outerHTML 作为模板内容
            template = getOuterHTML(el)
        }

        // template: 存储着最终用来生成渲染函数的字符串
        if (template) {
            /* istanbul ignore if */
            if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
                mark('compile')
            }

            // 获取转换后的render函数与staticRenderFns,并挂在$options上
            const { render, staticRenderFns } = compileToFunctions(template, {
                shouldDecodeNewlines,
                delimiters: options.delimiters,
                comments: options.comments
            }, this)

            options.render = render
            options.staticRenderFns = staticRenderFns

            /* istanbul ignore if */
            // 用来统计编译器性能, config是全局配置对象
            if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
                mark('compile end')
                measure(`vue ${this._name} compile`, 'compile', 'compile end')
            }
        }
    }

    // 调用之前说的公共mount方法
    // 重写$mount方法是为了添加模板编译的功能
    return mount.call(this, el, hydrating)
}
```  
首先判断`option.render`是否存在，如果存在，则直接调用`mount.call`，不存在，则通过优先考虑`option.template`属性，其次考虑el，最终将模版赋值给`template`，然后调用`compileToFunctions`来生成render函数。 最后调用`mount`，即原来不带编译功能当`Vue.prototype.$mount`。并调用`mountComponent`来将模版真正绑定到组件上。

`src/platforms/web/runtime/index.js`
```javascript
// el: 可以是一个字符串或者Dom元素
// hydrating 是Virtual DOM 的补丁算法参数
Vue.prototype.$mount = function (
    el?: string | Element,
    hydrating?: boolean
): Component {
    // 判断el, 以及宿主环境, 然后通过工具函数query重写el。
    el = el && inBrowser ? query(el) : undefined
    // 执行真正的挂载并返回
    return mountComponent(this, el, hydrating)
}
```

`src/core/instance/lifecycle.js`  

```javascript
export function mountComponent (
    vm: Component,    // 组件实例vm
    el: ?Element,     // 挂载点
    hydrating?: boolean
): Component {
    // 在组件实例对象上添加$el属性
    // $el的值是组件模板根元素的引用
    vm.$el = el
    if (!vm.$options.render) {
        // 如果不存在render，直接创建一个空的VNode
        vm.$options.render = createEmptyVNode
    }

    // 触发 beforeMount 生命周期钩子
    callHook(vm, 'beforeMount')


    let updateComponent   // 把渲染函数生成的虚拟DOM渲染成真正的DOM
    
    ...

    updateComponent = () => {
        vm._update(vm._render(), hydrating)
    }

    // 会在new Watcher的时候通过get方法执行一次
    // 也就是会触发第一次Dom的更新
    vm._watcher = new Watcher(vm, updateComponent, noop)
    hydrating = false

    // manually mounted instance, call mounted on self
    // mounted is called for render-created child components in its inserted hook
    if (vm.$vnode == null) {
        vm._isMounted = true
        callHook(vm, 'mounted')
    }
    return vm
}

```

这里用到了前面起到的 `render-watcher`，当`watcher`执行`update`的时候，会调用`updateComponent`，触发DOM更新。
> * vm._render 函数的作用是调用 vm.$options.render 函数并返回生成的虚拟节点(vnode)。template => render => vnode  
> * vm._update 函数的作用是把 vm._render 函数生成的虚拟节点 vnode 渲染成真正的 DOM。 vnode => real dom node

下面看一下`vm.$options.render`，如果options中已经定义render，则不作处理，如未定义，上文提到，则用`compileToFunctions`来将`compileToFunctions`转换成render函数与staticRenderFns,并挂在`$options`上。

`src/platforms/web/compiler/index.js`  

```javascript
const { compile, compileToFunctions } = createCompiler(baseOptions)
```  

`src/compiler/index.js` 

```javascript
export const createCompiler = createCompilerCreator(function baseCompile (
    template: string,
    options: CompilerOptions
): CompiledResult {
    // 解析html字符串的标签、元素、文本、注释等，生成AST(抽象语法树)
    const ast = parse(template.trim(), options)

    // 将AST进行静态优化
    optimize(ast, options)

    // 由AST生成render
    const code = generate(ast, options)
    
    return {
        ast,
        render: code.render,
        staticRenderFns: code.staticRenderFns
    }
})

```


