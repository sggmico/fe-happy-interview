## computed VS methods

> 面试题：computed 和 methods 有什么区别？

### 简单比较

1. computed为属性，可赋值，可设置 getter/setter，不能传参数； methods为方法，需要方法调用，可以传多个参数。
2. computed具有缓存特性，methods没有。

### 深入分析（源码）

> **computed** 和 **methods** 的初始化发生在  `beforeCreate`  —— `created` 期间的，具体到源码是在 `initState` 方法中来分别实现。下面分别来分析具体过程。

#### **methods** 

初始化过程源码

```javascript
function initMethods(vm: Component, methods: Object) {
    // ...
    for (const key in methods) {
      //...
      vm[key] = typeof methods[key] !== 'function' ? noop : bind(methods[key], vm)
    }
}
```

通过上面核心代码实现， **methods**初始化过程很简单。将其内部的声明的全部方法都通过 `bind` 函数绑定到了当前的组件实例上，并且将绑定this之后生成的**新函数**，**同名**挂载到了组件实例上。

#### **computed** 

初始化过程相对复杂，核心需要做两件事情：

1. 为每个计算属性创建一个watcher
   1. 将计算属性的getter传递给watcher来管理
   2. watcher的扩展配置中 添加 `lazy: true` ，用于辅助实现缓存特性。
2. 通过 `Object.defineproperty ` 为每个计算属性定义getter 和 setter
   1. getter 核心逻辑：当计算属性依赖的数据发生变化后，其对应 **watcher** 的 `dirty: true` ，说明计算属性待重新计算，当前值为脏值。那么，如果在组件渲染时使用到了该计算属性，渲染时会触发计算属性getter，会重新计算计算属性，随后 将 `dirty: false` 、收集依赖，返回新的计算属性的值
   2. setter 核心逻辑：很简单，就是直接执行 user setter func 就可以。

#### 源码

```javascript
function initComputed(vm: Component, computed: Object) {
  // $flow-disable-line
  const watchers = vm._computedWatchers = Object.create(null)

  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
 
    // create internal watcher for the computed property.
    watchers[key] = new Watcher(
      vm,
      getter || noop,
      noop,
      computedWatcherOptions
    )

    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    }
  }
}

export function defineComputed(
  target: any,
  key: string,
  userDef: Object | Function
) {
  const shouldCache = !isServerRendering()
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key)
      : createGetterInvoker(userDef)
    sharedPropertyDefinition.set = noop
  } else {
    sharedPropertyDefinition.get = userDef.get
      ? shouldCache && userDef.cache !== false
        ? createComputedGetter(key)
        : createGetterInvoker(userDef.get)
      : noop
    sharedPropertyDefinition.set = userDef.set || noop
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}

function createComputedGetter(key) {
  return function computedGetter() {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}

function createGetterInvoker(fn) {
  return function computedGetter() {
    return fn.call(this, this)
  }
}
```

#### 示例分析

```vue
	<template>
    <div class="app-wrapper">
        <button @click="(firstName = 'zhang'), (showName = !showName)">
            切换显示
        </button>
        <p v-if="showName">fullName: {{ fullName }}</p>
    </div>
</template>
<script>
export default {
    data() {
        return {
            firstName: "shi",
            lastName: "guoguo",
            showName: true,
        };
    },
    computed: {
        fullName() {
            console.log("computed getter");
            return this.firstName + " " + this.lastName;
        },
        // fullName: {
        //     get() {
      	//				console.log("computed getter");
        //        console.log("fullName getter");
        //        return this.firstName + " " + this.lastName;
        //     },
        // },
    },
    created() {
        console.log(this);
    },
};
</script>
```

上面示例，当页面初始化后，会打印计算属性中的 `“computed getter”`  。当点击按钮切换隐藏后，为执行计算属性中的打印内容。因为此时页面上没有使用到计算属性 `fullName` ，所以不会触发 计算属性的 getter，即不会重新计算该属性的值。但是当点击按钮后，已经修改了 `firstName` 的 值，此时，计算属性的 watcher 的属性 `dirty` 被标记为 `true`。当第二次点击按钮，切换为显示后，计算属性被使用，并且因为 dirty: true，会被重新计算，所以会再次打印 `“computed getter”`，并且页面显示为 `“zhang guoguo”`。