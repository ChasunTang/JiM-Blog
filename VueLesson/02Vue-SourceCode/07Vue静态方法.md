---
title:  Vue-Extend Vue-Component
date: 2017-11-28
categories: vue
---

### 1 Vue构造函数一节分析了Vue对象上的属性和方法如何一步步挂载上去的，接下来分析这些方法的具体实现

#### 1.1 Vue.extend(option)

```javascript
export function initExtend (Vue: GlobalAPI) {
  /**
   * Each instance constructor, including Vue, has a unique
   * cid. This enables us to create wrapped "child
   * constructors" for prototypal inheritance and cache them.
   */
  Vue.cid = 0
  let cid = 1

  /**
   * Class inheritance
   */
  Vue.extend = function (extendOptions: Object): Function {
    extendOptions = extendOptions || {}
    const Super = this
    const SuperId = Super.cid
    const cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {})
    if (cachedCtors[SuperId]) {
      return cachedCtors[SuperId]
    }

    const name = extendOptions.name || Super.options.name
    if (process.env.NODE_ENV !== 'production') {
      if (!/^[a-zA-Z][\w-]*$/.test(name)) {
        warn(
          'Invalid component name: "' + name + '". Component names ' +
          'can only contain alphanumeric characters and the hyphen, ' +
          'and must start with a letter.'
        )
      }
    }

    const Sub = function VueComponent (options) {
      this._init(options)
    }
    Sub.prototype = Object.create(Super.prototype)
    Sub.prototype.constructor = Sub
    Sub.cid = cid++
    Sub.options = mergeOptions(
      Super.options,
      extendOptions
    )
    Sub['super'] = Super

    // For props and computed properties, we define the proxy getters on
    // the Vue instances at extension time, on the extended prototype. This
    // avoids Object.defineProperty calls for each instance created.
    if (Sub.options.props) {
      initProps(Sub)
    }
    if (Sub.options.computed) {
      initComputed(Sub)
    }

    // allow further extension/mixin/plugin usage
    Sub.extend = Super.extend
    Sub.mixin = Super.mixin
    Sub.use = Super.use

    // create asset registers, so extended classes
    // can have their private assets too.
    ASSET_TYPES.forEach(function (type) {
      Sub[type] = Super[type]
    })
    // enable recursive self-lookup
    if (name) {
      Sub.options.components[name] = Sub
    }

    // keep a reference to the super options at extension time.
    // later at instantiation we can check if Super's options have
    // been updated.
    Sub.superOptions = Super.options
    Sub.extendOptions = extendOptions
    Sub.sealedOptions = extend({}, Sub.options)

    // cache constructor
    cachedCtors[SuperId] = Sub
    //这里就是调用Vue.extend(option)之后的返回值
    return Sub
  }
}

function initProps (Comp) {
  const props = Comp.options.props
  for (const key in props) {
    proxy(Comp.prototype, `_props`, key)
  }
}

function initComputed (Comp) {
  const computed = Comp.options.computed
  for (const key in computed) {
    defineComputed(Comp.prototype, key, computed[key])
  }
}

```

来看下这个时候Sub构造函数上有哪些方法和属性

* 静态属性

```javascript
Sub.cid 
Sub.options
Sub.extend
Sub.mixin
Sub.use
Sub.component
Sub.directive
Sub.filter

// 新增
Sub.super  // 指向父级构造函数
Sub.superOptions // 父级构造函数的options
Sub.extendOptions  // 传入的extendOptions
Sub.sealedOptions  // 保存定义Sub时，它的options值有哪些
```

* 原型属性

```javascript
Sub.prototype.constructor ==> Super(Vue)
Sub.__proto__ ==> Vue.prototype
//以下还有不可枚举和不可配置的一些属性
Sub.prototype.$props
Sub.prototype.$data
Sub.prototype.$ssrContext
Sub.prototype.$isServer
```

#### 1.2 Vue.component(option)

```javascript
export function initAssetRegisters (Vue: GlobalAPI) {
  /**
   * Create asset registration methods.
   */
  ASSET_TYPES.forEach(type => {
    Vue[type] = function (
      id: string,
      definition: Function | Object
    ): Function | Object | void {
      if (!definition) {
        //	这里可以通过 Vue.component('my-component')来获取注册的组件
        return this.options[type + 's'][id]
      } else {
        /* istanbul ignore if */
        if (process.env.NODE_ENV !== 'production') {
          if (type === 'component' && config.isReservedTag(id)) {
            warn(
              'Do not use built-in or reserved HTML elements as component ' +
              'id: ' + id
            )
          }
        }
        if (type === 'component' && isPlainObject(definition)) {
          //===========================
          //主要是下面这两行代码，可以看到Vue.component(id,option),最终还是调用的Vue.extend(option)
          //其中option中增加了name属性键值对
          definition.name = definition.name || id
          definition = this.options._base.extend(definition)
          //===========================
        }
        if (type === 'directive' && typeof definition === 'function') {
          definition = { bind: definition, update: definition }
        }
        //无论我们传入Vue.component(id,defination)中的defination是一个纯对象or一个函数，都会执行到这里，这里的this指的是Vue构造函数，将我们传递进来的defination给到Vue.options[type]
        //其中type可以是component ,filter directive
        //当我们全局注册之后，然后注册在Vue.options中的components组件在new Vue(options)的时候，会将Vue.options上的组件给到实例对象的vm.$options.components;
        this.options[type + 's'][id] = definition //这个this指的是Vue构造函数
        return definition
      }
    }
  })
}

```

### 2 接下来我们分析下使用以及所谓的全局注册组件和局部注册组件是如何实现的

* 全局注册:对于全局注册的组件，可以在任意Vue的实例中使用，因为全局注册的组件挂载在Vue.options.components上

```html
	<div id="box">
        <myComp></myComp>
    </div>
	<div id="box2">
        <myComp></myComp>
    </div>
    <script>
        var myCopm = Vue.extend({
            data() {
                return {
                    msg: '我是标题^^'
                };
            },
            methods: {
                change() {
                    this.msg = 'changed'
                }
            },
            template: '<h3 @click="change">{{msg}}</h3>'
        });
//会在Vue.options.components数组上增加一个aaa
        var ret = Vue.component('myComp', myComp);
        console.dir(myComp);
        console.dir(ret);
      //发现以上两个输出都是Sub构造函数
        console.dir(Vue);
      //在这里我们可以去输出里看下。Vue.options.components中多了 myComp
      //这也就实现了全局注册一个Vue组件，然后在
     /** Vue.options = {
            components: {KeepAlive,transition,transitionGroup,myComp}
            directives: {},
            filters: {},
            _base: Vue
         }
	  */
      //全局注册的组件在以下两个Vue的实例上都可以访问到
        var vm = new Vue({
            el: '#box',
            data: {
                bSign: true
            }
        });
      var vm2 = new Vue({
        el:'#box2'
      })
    </script>
```

* 局部注册

```html
<div id="box">
        <myComp></myComp>
  		<my-component></my-component>
    </div>
    <!--
	<div id='box2'>
       不能使用myComp组件，因为myComp是一个局部组件，它属于#box
        <myComp></myComp>
    </div>
	-->
    <script>
      //声明一个子组件
        var myComp = Vue.extend({
            data() {
                return {
                    msg: '我是标题^^'
                };
            },
            methods: {
                change() {
                    this.msg = 'changed'
                }
            },
            template: '<h3 @click="change">{{msg}}</h3>'
        });

        var vm = new Vue({
            el: '#box',
            data: {
                bSign: true
            },
            components:{
                'myComp':myComp,
              	'my-component': {
                    template: '<div>children component!</div>'
                }
            }
        });
      console.log(vm);
      //这里在vm.$option.components上可以看到{myComp:f VueComponent(options)}
      //所以这个就是局部注册的组件
        /**var vm2 = new Vue({
            el:"#box2"
        })*/
    </script>
```

### 3 Vue.filter 的实现

```javascript
export const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]
export function initAssetRegisters (Vue: GlobalAPI) {
  /**
   * Create asset registration methods.
   */
  ASSET_TYPES.forEach(type => {
    Vue[type] = function (
      id: string,
      definition: Function | Object
    ): Function | Object | void {
      if (!definition) {
        //	这里可以通过 Vue.filter('my-filter')来获取注册的过滤器
        return this.options[type + 's'][id]
      } else {
        /* istanbul ignore if */
        if (process.env.NODE_ENV !== 'production') {
          if (type === 'component' && config.isReservedTag(id)) {
            warn(
              'Do not use built-in or reserved HTML elements as component ' +
              'id: ' + id
            )
          }
        }
        if (type === 'component' && isPlainObject(definition)) {
          definition.name = definition.name || id
          definition = this.options._base.extend(definition)
        }
        if (type === 'directive' && typeof definition === 'function') {
          definition = { bind: definition, update: definition }
        }
      //this指的是Vue这个构造函数，Vue.filter(id,definition)会执行到这里，上面的if语句都不会进入；
        this.options[type + 's'][id] = definition
        return definition
      }
    }
  })
}
```

```javascript
var filters = {
  filter1:function(){
    console.log('filters1')
  },
  filter2:function(){
    console.log('filters1')
  },
  filter3:function(){
    console.log('filters1')
  }
};
Object.keys(filters).forEach(k => Vue.filter(k, filters[k]));
console.dir(Vue);//Vue.options.filters:{filter1,filter2,filter3}
```

### 4 Vue.use(plugin)

```javascript
export function initUse (Vue: GlobalAPI) {
  Vue.use = function (plugin: Function | Object) {
    const installedPlugins = (this._installedPlugins || (this._installedPlugins = []))
    if (installedPlugins.indexOf(plugin) > -1) {
      return this
    }

    // additional parameters 将arguments对象转化为数组，方便后面apply调用
    const args = toArray(arguments, 1)
    args.unshift(this)
    if (typeof plugin.install === 'function') {
      //这个apply用的真巧妙，install函数中只接受第一个参数Vue构造函数，apply将args数组拆分开传过去也不会用到；
      plugin.install.apply(plugin, args)
    } else if (typeof plugin === 'function') {
      plugin.apply(null, args)
    }
    installedPlugins.push(plugin)
    return this
  }
}
```

```javascript
Vue.use(VueRouter);
console.dir(Vue._installedPlugins);//[VueRouter]
```

### 5 Vue.mixin()

```javascript
export function initMixin (Vue: GlobalAPI) {
  Vue.mixin = function (mixin: Object) {
    this.options = mergeOptions(this.options, mixin)
    return this
  }
}
```

使得传入Vue.mixin(Object)中的Object对象融入Vue.options对象；

### 6 Vue.directive()

```javascript
export const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]
export function initAssetRegisters (Vue: GlobalAPI) {
  /**
   * Create asset registration methods.
   */
  ASSET_TYPES.forEach(type => {
    Vue[type] = function (
      id: string,
      definition: Function | Object
    ): Function | Object | void {
      if (!definition) {
        //	这里可以通过 Vue.directive('my-directive')来获取注册的指令
        return this.options[type + 's'][id]
      } else {
        /* istanbul ignore if */
        if (process.env.NODE_ENV !== 'production') {
          if (type === 'component' && config.isReservedTag(id)) {
            warn(
              'Do not use built-in or reserved HTML elements as component ' +
              'id: ' + id
            )
          }
        }
        if (type === 'component' && isPlainObject(definition)) {
          definition.name = definition.name || id
          definition = this.options._base.extend(definition)
        }
    //如果传给Vue.directive(id,definition) 中的definition是一个函数，则会走这里重新定义definition
        if (type === 'directive' && typeof definition === 'function') {
          definition = { bind: definition, update: definition }
        }
        //如果传给Vue.direction(id,definition)中的definition是一个对象，则直接走这里
        this.options[type + 's'][id] = definition
        return definition
      }
    }
  })
}
```

```javascript
// 注册一个全局自定义指令 `v-focus`
Vue.directive('focus', {
  // 当被绑定的元素插入到 DOM 中时……
  inserted: function (el) {
    // 聚焦元素
    el.focus()
  }
})
//Vue.directive(id,definition)的返回值就是处理后的definition;
```

### 7 Vue.set( )

[这里解释了Vue.set的存在原因](https://github.com/Ma63d/vue-analysis/issues/1)

我们考虑一下这样的情况，比如我的data:{a:{b:true}}，这个时候，如果页面有dom上有个指令`:class="a"`，而我想响应式的删除data.a的b属性，此时我就没有办法了，因为defineReactive中的getter/setter都不会执行(他们甚至还会在delete a.b时被清空)，闭包里的那个dep就无法通知对应的watcher。

**这就是getter和setter存在的缺陷：只能监听到属性的更改，不能监听到属性的删除与添加。**

参考[《Vue双向数据绑定实现原理2》](https://github.com/jimwmg/JiM-Blog/tree/master/VueLesson/02Vue-SourceCode)

Vue的解决办法是提供了响应式的api: vm.$set/vm.$delete/ Vue.set/ Vue.delete /数组的$set/数组的$remove。

observe/index.js

```javascript
export function set (target: Array<any> | Object, key: any, val: any): any {
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }
  const ob = (target: any).__ob__
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid adding reactive properties to a Vue instance or its root $data ' +
      'at runtime - declare it upfront in the data option.'
    )
    return val
  }
  if (!ob) {
    target[key] = val
    return val
  }
  defineReactive(ob.value, key, val)
  ob.dep.notify()
  return val
}

```

### 8 Vue.delete 

基本原理同Vue.set

```javascript
export function del (target: Array<any> | Object, key: any) {
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.splice(key, 1)
    return
  }
  const ob = (target: any).__ob__
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid deleting properties on a Vue instance or its root $data ' +
      '- just set it to null.'
    )
    return
  }
  if (!hasOwn(target, key)) {
    return
  }
  delete target[key]
  if (!ob) {
    return
  }
  ob.dep.notify()
}
```

