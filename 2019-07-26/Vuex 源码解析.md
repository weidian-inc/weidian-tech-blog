# Vuex 源码解析

[Vuex是什么？](https://vuex.vuejs.org/zh/)



## Vuex使用方式

1. 安装Vuex

```js
import Vue from 'vue'
import Vuex from 'vuex'
Vue.use(Vuex)
```

2. 创建store

```js
const stroe = new Vuex.Store({
  //store 内容
})
```

3. 注入store

```js
const app = new Vue({
  store,
  //vue 实例配置
})
```



接下来按照使用方式的步骤进行分析。

## Vuex.install——安装Vuex

[src/index.js](https://github.com/luoway/vuex/blob/dev/src/index.js)

```js
import { Store, install } from './store'
//import ...
export default {
  Store,
  install,
  version: '__VERSION__',
  mapState,
  mapMutations,
  mapGetters,
  mapActions,
  createNamespacedHelpers
}
```

根据[Vue.use()](https://cn.vuejs.org/v2/api/#Vue-use)用法说明，`Vue.use(Vuex)`执行时会调用`Vuex.install()`，内容如下：

[src/store.js](https://github.com/luoway/vuex/blob/dev/src/store.js#L499)

```js
import applyMixin from './mixin'
//...
let Vue // bind on install
//...
export function install (_Vue) {
  if (Vue && _Vue === Vue) {
    //省略开发环境提示
    return
  }
  Vue = _Vue
  applyMixin(Vue)
}
```

再看`applyMixin(Vue)`

[src/mixin.js](https://github.com/luoway/vuex/blob/dev/src/mixin.js)

```js
export default function (Vue) {
  //省略Vue 1.x、2.x版本区分
  Vue.mixin({ beforeCreate: vuexInit })

  /**
   * Vuex初始化钩子，注入到每个实例的初始化钩子列表。
   */

  function vuexInit () {
    const options = this.$options
    // store injection
    if (options.store) {
      this.$store = typeof options.store === 'function'
        ? options.store()
        : options.store
    } else if (options.parent && options.parent.$store) {
      this.$store = options.parent.$store
    }
  }
}
```

执行`Vue.use(Vuex)`目的就是在每个Vue实例初始化时，往`beforeCreate`生命周期中加入`vueInit`函数。

在“注入store”这一步：

```js
const app = new Vue({
  store,
  //vue 实例配置
})
```

执行Vue实例的`beforeCreate`生命周期，`vueInit`函数被执行，运行时`this`指向当前vue实例上下文，因此`this.$options`指向当前vue组件**配置对象**

> 配置对象 是指，`new Vue(options)`时传入的`options`，实例内可以通过`this.$options`访问到

传入的`store`可以被`options.store`访问到，此时执行

```js
this.$store = typeof options.store === 'function'
        ? options.store()
        : options.store
```

将`options.store`内容赋值给`this.$store`，即完成了`store`的注入。

而`options.store`不存在时，即没有给当前组件配置`store`时，`vueInit`会尝试访问父组件已经注入完成的`$store`，若存在则赋值给当前组件`this.$store`，实现[组件树](https://cn.vuejs.org/v2/guide/components.html#组件的组织)内共享`store`。



至此，Vuex的安装、注入看完了，接下来具体看创建Store。

## Vuex.Store(options)——创建Store

[src/store.js](https://github.com/luoway/vuex/blob/dev/src/store.js#L8)

```js
export class Store {
  constructor () {}

  get state () {}
  set state (v) {}
  commit () {}
  dispatch () {}
  //...
}
```

Vuex.Store是一个ES6 class，JS里是没有Class数据类型的，本质上还是个构造函数。



### 实例化过程

继续看Store类的实例化过程：

```js
//...
import ModuleCollection from './module/module-collection'
//...

let Vue // bind on install

export class Store {
  constructor (options = {}) {
    // 如果没有安装，则自动安装
    if (!Vue && typeof window !== 'undefined' && window.Vue) {
      install(window.Vue)
    }
    // 省略开发中的检测警告
    const {
      plugins = [],
      strict = false
    } = options
    
    // store内部状态
    //...
    this._wrappedGetters = Object.create(null)
    this._modules = new ModuleCollection(options)
    //...
    
    // bind commit and dispatch to self
    const store = this
    const { dispatch, commit } = this
    this.dispatch = function boundDispatch (type, payload) {
      return dispatch.call(store, type, payload)
    }
    this.commit = function boundCommit (type, payload, options) {
      return commit.call(store, type, payload, options)
    }
    
    // strict mode
    this.strict = strict
    
    const state = this._modules.root.state

    // 初始化根模块。
    // 递归地注册所有子模块
    // 并收集所有模块的getters到this._wrappedGetters
    installModule(this, state, [], this._modules.root)

    // 初始化 store vm, 负责实现响应变化
    // (也注册_wrappedGetters为computed属性)
    resetStoreVM(this, state)

    //...
  }
  
  get state () {
    return this._vm._data.$$state
  }
  set state (v) {
    if (process.env.NODE_ENV !== 'production') {
      assert(false, `use store.replaceState() to explicit replace store state.`)
    }
  }
  
  dispatch(){}
  commit(){}
  //...
}
```

上面省略过后的实例化过程中，主要做了五件事：

- 自动安装Vuex。即CDN引入vue.js时（`window.Vue`存在），可以省略手动调用`Vue.use(Vuex)`
- 初始化Store内部状态
- 绑定`commit`、`dispatch`指向当前Store实例
- `installModule`
- `resetStoreVM`

关于第二点“绑定commit、dispatch指向当前Store实例”，可以这么理解：

```js
const store = new Vuex.Store({
  state: {
    val: 0
  },
  mutations:{
    addVal(state){
      state.val += 1
    }
  }
})//store值为一个Store实例，内容为{state, dispath(), commit()}
const ct = store.commit
ct('addVal')
```

若未绑定`commit`方法的`this`指向，此时变量`ct`值为未绑定`this`指向的`commit`函数，当执行`ct('addVal')`时，`this`会指向当前（全局）作用域上下文`window`，`ct`函数内部使用`this`就不会按预期地指向`store`。因此需要对`commit`、`dispatch`进行绑定。



###resetStoreVM

`Vuex.Store`的基础是`state`，在`Store`类中定义为：

```js
	get state () {
    return this._vm._data.$$state
  }
  set state (v) {
    if (process.env.NODE_ENV !== 'production') {
      assert(false, `use store.replaceState() to explicit replace store state.`)
    }
  }
```

定义的是一个取值器和一个赋值器，是一个伪属性，没有真实值。

当尝试直接修改state时，

```js
this.$store.state = newState
```

修改不会生效，而是执行赋值器方法，提示使用`store.replaceState()`方法。

当读取`state`时，读取的实际上是`this._vm._data.$$state`，`this._vm`在`resetStoreVM()`中赋值，下面具体看[resetStoreVM源码](https://github.com/luoway/vuex/blob/dev/src/store.js#L250)

```js
//...
import { forEachValue, isObject, isPromise, assert, partial } from './util'
//...
function resetStoreVM (store, state, hot) {
  //...
  // bind store public getters
  store.getters = {}
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  forEachValue(wrappedGetters, (fn, key) => {
    computed[key] = partial(fn, store)
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })

  // 使用一个Vue实例来存储状态树
  // 抑制警告，以防用户添加了某些全局mixins
  const silent = Vue.config.silent
  Vue.config.silent = true
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
  Vue.config.silent = silent

  //...
}
```

resetStoreVM函数的关键部分是

```js
store._vm = new Vue({
  data: {
    $$state: state
  },
  computed
})
```

其中`state`是`resetStoreVM`的入参，`computed`对应`store._wrappedGetters`，并在`store.getters`中定义取值器，获取`_vm`的计算结果。

> Vuex 是一个**专为 Vue.js 应用程序开发**的状态管理模式。

这就是专用的原因：数据响应更新是借由Vue实现的。



需要指出一处特殊的地方：这里的配置对象中data是一个对象，而非[通常的函数](https://cn.vuejs.org/v2/guide/components.html#data-必须是一个函数)。

这仅仅是因为不需要对Vue实例进行复用，直接赋值对象性能较好。

另一个需要理解的地方是：`this._vm._data.$$state`从`_data`中访问`$$state`。

`_data`是Vue实例`_vm`的真实值（见[Vue源码](https://github.com/vuejs/vue/blob/dev/src/core/instance/state.js#L114)），而`_vm.$$state`是对`_data.$$state`取值的伪属性（见[Vue源码](https://github.com/vuejs/vue/blob/dev/src/core/instance/state.js#L147)）。同样是因为直接取值性能较好。



至此，可以明确`this.$store.state`访问的结果是由`state`、`_wrappedGetters`构成的Vue实例的状态。

接下来看`state`是如何赋值的。

### ModuleCollection

Vuex.Store构造函数实例化过程中可以找到state初始赋值的位置：

```js
import ModuleCollection from './module/module-collection'
//...
export class Store {
  constructor(options = {}){
    //...
    this._modules = new ModuleCollection(options)
    //...
    const state = this._modules.root.state
    installModule(this, state, [], this._modules.root)
    //...
  }
  //...
}
//...
```

赋予的值为`this._modules.root.state`，语义上理解为“根模块的状态”。`this._modules`是一个“模块集合”类的实例，具体看`ModuleCollection`内容：

[src/module/module-collection.js](https://github.com/luoway/vuex/blob/dev/src/module/module-collection.js#L4)

```js
import Module from './module'
//...

export default class ModuleCollection {
  constructor (rawRootModule) {
    // 注册根模块 (Vuex.Store options)
    this.register([], rawRootModule, false)
  }
  
  //...
  
  register (path, rawModule, runtime = true) {
    // 省略开发环境类型断言

    const newModule = new Module(rawModule, runtime)
    //...

    // 注册嵌套模块
    if (rawModule.modules) {
      forEachValue(rawModule.modules, (rawChildModule, key) => {
        this.register(path.concat(key), rawChildModule, runtime)
      })
    }
  }
  
  //...
}

//...
```

可以看出模块集合ModuleCollection中的模块Module内容，是在Module类中定义的。

进一步查看Module类的源码：

[src/module/module.js](https://github.com/luoway/vuex/blob/dev/src/module/module.js#L4)

```js
import { forEachValue } from '../util'

// 存储模块的基本数据结构，包含一些属性和方法
export default class Module {
  constructor (rawModule, runtime) {
    this.runtime = runtime
    // 存储子项
    this._children = Object.create(null)
    // 存储开发者传入的原始模块对象
    this._rawModule = rawModule
    const rawState = rawModule.state

    // 存储原始模块的状态
    this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
  }

  get namespaced () {
    return !!this._rawModule.namespaced
  }

  addChild (key, module) {
    this._children[key] = module
  }

  removeChild (key) {
    delete this._children[key]
  }

  getChild (key) {
    return this._children[key]
  }

  update (rawModule) {
    this._rawModule.namespaced = rawModule.namespaced
    if (rawModule.actions) {
      this._rawModule.actions = rawModule.actions
    }
    if (rawModule.mutations) {
      this._rawModule.mutations = rawModule.mutations
    }
    if (rawModule.getters) {
      this._rawModule.getters = rawModule.getters
    }
  }

  forEachChild (fn) {
    forEachValue(this._children, fn)
  }

  forEachGetter (fn) {
    if (this._rawModule.getters) {
      forEachValue(this._rawModule.getters, fn)
    }
  }

  forEachAction (fn) {
    if (this._rawModule.actions) {
      forEachValue(this._rawModule.actions, fn)
    }
  }

  forEachMutation (fn) {
    if (this._rawModule.mutations) {
      forEachValue(this._rawModule.mutations, fn)
    }
  }
}
```

可见`Module`类负责存储

- `runtime`标识
- 子项的索引
- 当前模块的状态

并提供

- 原始模块`namespaced`的布尔值取值器
- 增删查子项的方法
- 更新原始模块（更新内容包括四个字段：`namespaced`、`actions`、`mutations`、`getters`）的方法
- 遍历原始模块的子项、`actions`、`mutations`、`getters`的方法

回过头看`ModuleCollection`类：

```js
import Module from './module'
//...

export default class ModuleCollection {
  constructor (rawRootModule) {
    // 注册根模块 (Vuex.Store options)
    this.register([], rawRootModule, false)
  }

  get (path) {
    return path.reduce((module, key) => {
      return module.getChild(key)
    }, this.root)
  }

  getNamespace (path) {
    let module = this.root
    return path.reduce((namespace, key) => {
      module = module.getChild(key)
      return namespace + (module.namespaced ? key + '/' : '')
    }, '')
  }

  update (rawRootModule) {
    update([], this.root, rawRootModule)
  }

  register (path, rawModule, runtime = true) {
    // 省略开发环境类型断言

    const newModule = new Module(rawModule, runtime)
    if (path.length === 0) {
      this.root = newModule
    } else {
      const parent = this.get(path.slice(0, -1))
      parent.addChild(path[path.length - 1], newModule)
    }

    // 注册嵌套模块
    if (rawModule.modules) {
      forEachValue(rawModule.modules, (rawChildModule, key) => {
        this.register(path.concat(key), rawChildModule, runtime)
      })
    }
  }

  unregister (path) {
    const parent = this.get(path.slice(0, -1))
    const key = path[path.length - 1]
    if (!parent.getChild(key).runtime) return

    parent.removeChild(key)
  }
}

function update (path, targetModule, newModule) {
  // 省略开发环境类型断言

  // 更新目标模块
  targetModule.update(newModule)

  // 更新嵌套模块
  if (newModule.modules) {
    for (const key in newModule.modules) {
      if (!targetModule.getChild(key)) {
        // ...
        return
      }
      update(
        path.concat(key),
        targetModule.getChild(key),
        newModule.modules[key]
      )
    }
  }
}
```

上面源码中`path`是指嵌套模块的路径，形如`a/b/c`这样的嵌套模块，对应的`path`值为`[a, b, c]`。

可见`ModuleCollection`类负责存储

- 根模块`root`

并提供

- `get(path)`：根据路径，获取根模块中某个模块
- `getNamespace(path)`：根据路径，获取模块命名空间
- `update()`：递归地更新根模块及其后代的原始模块
- `register()`：递归地注册模块及其嵌套模块，注册模块是指将传入的原始模块存储为`Module`实例
- `unregister(path)`：取消注册模块，即将指定路径的模块从父级的子项索引中移除

现在可以明确在[src/store.js](https://github.com/luoway/vuex/blob/dev/src/store.js#L52)中

```js
const state = this._modules.root.state
installModule(this, state, [], this._modules.root)
resetStoreVM(this, state)
```

初始赋值的`state`是指，`ModuleCollection`实例根模块的状态。这个状态仅包括根模块，不含嵌套模块，这样的`state`传递给`resetStoreVM`不足以实现嵌套模块的数据响应。

下面看`installModule()`内容。

### installModule

[src/store.js](https://github.com/luoway/vuex/blob/dev/src/store.js#L298)

```js
function installModule (store, rootState, path, module, hot) {
  const isRoot = !path.length
  const namespace = store._modules.getNamespace(path)

  // 注册到命名空间映射表
  if (module.namespaced) {
    // 省略重复注册警告
    store._modulesNamespaceMap[namespace] = module
  }

  // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, module.state)
    })
  }
  
  //...

  module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
  })
}

function getNestedState (state, path) {
  return path.length
    ? path.reduce((state, key) => state[key], state)
    : state
}
```

`installModule()`递归地完成了模块的安装，其中就包括`state`嵌套赋值。

`store._withCommit()`是定义在`Store`类上的方法（见[源码](https://github.com/luoway/vuex/blob/dev/src/store.js#L218)）：

```js
export class Store {
  constructor (options = {}) {
    //...
    this._committing = false
    //...
  }
  //...
  _withCommit (fn) {
    const committing = this._committing
    this._committing = true
    fn()
    this._committing = committing
  }
}
```

该方法修改`store._committing`的目的只有一个

```js
import { assert } from './util'
//...
function enableStrictMode (store) {
  store._vm.$watch(function () { return this._data.$$state }, () => {
    if (process.env.NODE_ENV !== 'production') {
      assert(store._committing, `do not mutate vuex store state outside mutation handlers.`)
    }
  }, { deep: true, sync: true })
}
```

其中，`assert`在第一个参数为假值时，才会抛出错误。

[src/util.js](https://github.com/luoway/vuex/blob/dev/src/util.js)

```js
export function assert (condition, msg) {
  if (!condition) throw new Error(`[vuex] ${msg}`)
}
```

即Vuex在严格模式下，当`this._data.$$state`被观察到发生变化时，会在开发环境给出警告，而通过`_withCommit`包装后执行函数，则不会发出警告。

同理，Vuex `mutations`中状态变化不会发出警告，也是调用了`_withCommit`方法

下面就来分析`Store.commit()`方法：

### commit

[源码位置](https://github.com/luoway/vuex/blob/dev/src/store.js#L82)

```js
//...
export class Store {
  //...
  commit (_type, _payload, _options) {
    // 检查对象风格的提交方式
    const {
      type,
      payload,
      options
    } = unifyObjectStyle(_type, _payload, _options)

    const mutation = { type, payload }
    const entry = this._mutations[type]
    if (!entry) {
      // 省略警告
      return
    }
    this._withCommit(() => {
      // 执行指定的mutation函数
      entry.forEach(function commitIterator (handler) {
        handler(payload)
      })
    })
    this._subscribers.forEach(sub => sub(mutation, this.state))
    //...
  }
  //...
}

//...

function unifyObjectStyle (type, payload, options) {
  if (isObject(type) && type.type) {
    options = payload
    payload = type
    type = type.type
  }

  if (process.env.NODE_ENV !== 'production') {
    assert(typeof type === 'string', `expects string as the type, but found ${typeof type}.`)
  }

  return { type, payload, options }
}
//...
```

首先，使用`unifyObjectStyle()`函数对入参进行转换，用以支持[对象风格的提交方式](https://vuex.vuejs.org/zh/guide/mutations.html#对象风格的提交方式)。

然后，通过`this._withCommit()`包装执行入参`type`指定的mutation函数，可以避免严格模式警告。

最后，触发`subscribers`订阅器，这些订阅器是通过[subscribe() API](https://vuex.vuejs.org/zh/api/#subscribe)添加的。

类似的，`Store.dispatch()`方法也很简单：

### dispatch

[源码位置](https://github.com/luoway/vuex/blob/dev/src/store.js#L116)

```js
export class Store {
  //...
  
  dispatch (_type, _payload) {
    // 检查对象风格的提交方式
    const {
      type,
      payload
    } = unifyObjectStyle(_type, _payload)

    const action = { type, payload }
    const entry = this._actions[type]
    if (!entry) {
      // 省略警告
      return
    }

    //省略_actionSubscribers

    const result = entry.length > 1
      ? Promise.all(entry.map(handler => handler(payload)))
      : entry[0](payload)

    return result.then(res => {
      //省略_actionSubscribers
      return res
    })
  }
	//...
}
```

与`commit`不同的是，`dispatch`中执行的`_actions`是Promise实例，是在`installModule()`中创建的

[源码位置](https://github.com/luoway/vuex/blob/dev/src/store.js#L329)

```js
function installModule (store, rootState, path, module, hot) {
  //...
  module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store, type, handler, local)
  })
  //...
}
```

[registerAction()源码位置](https://github.com/luoway/vuex/blob/dev/src/store.js#L429)

```js
function registerAction (store, type, handler, local) {
  const entry = store._actions[type] || (store._actions[type] = [])
  entry.push(function wrappedActionHandler (payload, cb) {
    let res = handler.call(store, {
      dispatch: local.dispatch,
      commit: local.commit,
      getters: local.getters,
      state: local.state,
      rootGetters: store.getters,
      rootState: store.state
    }, payload, cb)
    if (!isPromise(res)) {
      res = Promise.resolve(res)
    }
    //省略devtool
    return res
  })
}
```



至此，Vuex常用部分的源码已经解析完成，仍有一些相对独立部分的源码没有提到，请自行了解。

## 总结

用思维导图总结一下本文解析的源码内容：

![Vuex](https://raw.githubusercontent.com/luoway/vuex/dev/imgs/Vuex.png)

![ModuleCollection](https://raw.githubusercontent.com/luoway/vuex/dev/imgs/ModuleCollection.png)

在阅读过源码后，重新回答Vuex官网上提出的这个问题：[什么情况下我应该使用 vuex？](https://vuex.vuejs.org/zh/#什么情况下我应该使用-vuex？)

### 什么情况下我应该使用 Vuex？

当你需要在Vue组件中管理共享状态时，就可以使用Vuex。

Vuex不仅提供了基础的状态管理、提交改动、分发事件，还提供了以嵌套模块、命名空间的方式来分治模块的模块化方案，更进一步提供了本文没有提及的插件（plugins）、监听、订阅、动态注册等功能。

即使你只用到了Vuex其中的一部分能力，Vuex提供的丰富能力意味着良好的扩展性，这对项目维护而言是有益的。

那为什么还需要思考这个该不该用的问题呢？因为这是一个技术选型问题，主要影响因素不在于Vuex技术本身，而在于项目本身。该不该用的问题，应当换一种问法：

> 可以使用Vuex的Vue项目，为什么不用Vuex？

我的回答是：

- 性价比考量

  项目的状态管理是否复杂，或预期能够复杂到值得引入压缩后超过9KB的Vuex？开发时间有没有压力？

- 有没有其他选择

  提供状态管理的库不止Vuex一种，是否有更熟悉、更亲睐的其他方案？比如团队内部自研库。

- 性能问题

  Vuex源码多处使用递归，嵌套的模块越多，嵌套层级越深，性能越差，这种情况下不得不考虑不使用Vuex的更高性能状态管理方案。