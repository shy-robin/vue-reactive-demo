# Vue响应式原理

## 流程图

![](https://gitee.com/gainmore/imglib/raw/master/img/20210828225342.png)

从流程图可以看出，Vue 的响应式原理设计了五个类：`Vue`、`Observer`、`Compile`、`Dep`、`Watcher`。

## Vue

实例化一个 Vue 对象是使用 Vue 的第一步。

Vue 的初始化需要给 Vue 传入一个对象，形如：

```js
const vm = new Vue({
    el: '#app',
    data: {
        name: 'xxx',
        info: {}
    },
    created() {
		//...
    }
})
```

其中，`el` 和 `data` 这两个属性是必不可少的。

- el 指定了 Vue 作用的区域；
- data 制定了 Vue 双向绑定的数据。

这两个属性分别传给 `Compile` 和 `Observer` 这两个类去执行。

```js
class Vue {
  constructor(options) {
    // 1.保存数据
    this.$options = options
    this.$data = options.data

    // 2.将 data 添加到响应式中
    new Observer(this.$data)

    // 3.代理 this.$data 的数据（可以用 vm.xxx 的形式调用 data 里的属性）
    Object.keys(this.$data).forEach(key => {
      this._proxy(key)
    })

    // 4.处理 el
    new Compile(options.el, this)
  }

  _proxy(key) {
    Object.defineProperty(this, key, {
      configurable: true, // 可以被删除
      enumerable: true, // 属性可以被遍历
      get() {
        return this.$data[key]
      },
      set(newVal) {
        this.$data[key] = newVal
      }
    })
  }
}
```



## Observer

```js
class Observer {
  constructor(data) {
    this.data = data

    this.observe(this.data)
  }

  observe(data) {
    if (!data || typeof data !== 'object') {
      return
    }
    Object.keys(data).forEach(key => {
      this.defineReactive(data, key, data[key])
    })
  }

  defineReactive(data, key, val) {
    this.observe(val) // 递归调用，给所有是对象的子元素添加响应式

    const dep = new Dep() // 为每个属性创建一个订阅器
    Object.defineProperty(data, key, {
      enumerable: true, // 属性可以被遍历
      configurable: true, // 属性可以被删除
      get() {
        // 如果缓存有值则将缓存的 Watcher 加入 dep，
        // 因为 Watcher 的 update 会触发 getter，利用缓存可以避免重复添加 Watcher 到 dep
        if (Dep.target) {
          dep.addSub(Dep.target)
        }
        return val
      },
      set(newVal) {
        if (val === newVal) { // 如果新值等于旧值，不添加响应式
          return
        }
        val = newVal
        dep.notify() // 属性改变，通知订阅器更新
      }
    })
  }
}
```



## Dep

```js
// 订阅器（收集订阅者 + 通知订阅者更新）
class Dep {
  constructor() {
    this.subs = [] // 依赖的 Watcher 列表
  }
  addSub(watcher) {
    this.subs.push(watcher)
  }
  notify() {
    this.subs.forEach(watcher => {
      watcher.update() // 更新每个 Watcher
    })
  }
}
```



## Compile

```js
// 对 html 文档进行模板编译
class Compile {
  constructor(el, vm) {
    vm.$el = document.querySelector(el) // 真实 DOM
    this.vm = vm // vue 实例

    // 创建文档片段（放在内存中），把所有 DOM 操作放到文档片段中，
    // 操作完成之后重新加入到文档流中，减少回流和重绘，提高 DOM 操作的性能。
    const fragment = document.createDocumentFragment()

    // 把所有子节点都加入到文档片段中（即将原文档放到内存中）
    let childNode
    while (childNode = vm.$el.firstChild) {
      fragment.appendChild(childNode)
    }

    // 进行模板编译
    this.replace(fragment)

    // 将文档片段加到文档中（即将文档重新放到原文档中）
    vm.$el.appendChild(fragment)
  }
  
  // 
  replace(node) {
    // 匹配插值表达式，\s 表示空格，\S 表示非空格
    const reg = /\{\{\s*(\S+)\s*\}\}/

    // 如果 node 是文本，则进行文本替换，并且停止递归
    if (node.nodeType === 3) {
      // 获取文本内容
      const text = node.textContent
      // 正则匹配和提取
      const rst = reg.exec(text)
      // 如果匹配成功
      if (rst) {
        // 如果有 `info.name` 这类属性，则需要拆分，然后使用 reduce 转换为 vm[info][name] 调用
        const value = rst[1].split('.').reduce((v, key) => v[key], this.vm)
        // 将文本中的插值替换成值（初始渲染）
        node.textContent = text.replace(reg, value)
        // 创建 watcher 实例
        new Watcher(this.vm, rst[1], newVal => {
          // 闭包，引用外层函数的变量，使变量持续保存在内存中
          node.textContent = text.replace(reg, newVal)
        })
      }
      return
    }

    // v-model 双向数据绑定
    // 如果当前节点是 input 输入框
    if (node.nodeType === 1 && node.tagName.toUpperCase() === 'INPUT') {
      // 1.data 数据变化使输入框数据更新
      const attrs = Array.from(node.attributes) // 判断输入框是否有 v-model 属性
      const rst = attrs.find(item => item.name === 'v-model')
      if (rst) {
        // v-model 的值
        const key = rst.value
        const value = key.split('.').reduce((v, k) => v[k], this.vm)
        node.value = value // 初始化数据

        // 创建 watcher 实例
        new Watcher(this.vm, key, newVal => {
          node.value = newVal // 更新数据
        })
        
        // 2.输入框数据变化使 data 数据更新
        node.addEventListener('input', event => {
          const keyArr = key.split('.') // 属性名数组
          let len = keyArr.length
          const value = event.target.value // input 的内容
          const obj = keyArr.slice(0, len - 1).reduce((v, k) => v[k], this.vm)
          obj[keyArr[len - 1]] = value // 给 data 属性赋值，触发 setter
        })
      }

    }

    // 如果不是文本节点，可能是一个 DOM 元素，则需要进行递归
    node.childNodes.forEach(childNode => this.replace(childNode))
  }
}
```



## Watcher

```js
// 订阅者（更新视图）
class Watcher {
  // vm: Vue 实例，里面保存了所有 data
  // key: 属性名，通过 key 可以找到对应的属性
  // cb: 当前 watcher 如何更新自己的文本内容的回调函数
  constructor(vm, key, cb) {
    this.vm = vm
    this.key = key
    this.cb = cb

    // 初始化
    Dep.target = this // 1.把当前 watcher 缓存起来
    this.update() // 2.更新触发 getter，因为缓存中有值，所以会把 watcher 加入 dep
    Dep.target = null // 3.清空缓存
  }

  update() { // 更新（直接调用 watcher 的 update 不会将 watcher 加入 dep）
    const value = this.key.split('.').reduce((v, k) => v[k], this.vm) // 会触发 getter
    this.cb(value)
  }
}
```



## 总结

- 初始化 Vue 是需要传入一个 options 对象，这个对象中必须包含 el 和 data 两个参数；
- 其中，data 会传入到 Observer 类，el 会被传入到 Compile 类；
- 在 Observer 中会递归每个 data 属性，并通过 defineReactive 方法给每个属性添加响应式，其中会给每个属性设置 getter、setter，然后创建一个 Dep 订阅器，用于收集所有依赖的 watcher 实例；
- 在 Compile 中会根据 el 这个选择器，获取到对应的 DOM，然后创建一个 documentFragment，将 DOM 里的节点都加入进 fragment 中；
- 之后会对 fragment 进行模板编译解析，通过递归每个节点，利用正则表达式匹配出插值表达式，然后获取到插值表达式对应的属性；
- 每获取到一个插值表达式就需要创建一个 Watcher 实例，创建实例需要传入三个参数：vue 实例、插值表达式对应的属性名、更新 DOM 的回调函数；
- Watcher 实例被创建时，就会调用自身的 update 函数，update 函数会获取 Vue 中 data 的属性，所以会触发 getter，因此会利用 Dep 的 addSub 方法将 watcher 实例添加到对应的订阅器 Dep 中，同时会调用更新回调函数，此时便对页面进行了初始化；
- 之后，如果改变了 data 中的值，就会触发响应式数据的 setter，于是便会调用 Dep 的 notify 通知每个 watcher，之后每个 watcher 会调用 update 函数，update 函数内部又会调用绑定的更新回调函数使页面更新。

**上述过程只是实现了 data -> view 的单向数据绑定，即 data 数据修改会触发 view 视图层更新；但是 view 视图更新却不能触发 data 更新。**

要实现 view -> data 的绑定更加简单，只需要给 view 的 DOM 元素添加监听事件，比如给输入框添加 input 事件，监听用户的输入，然后再做一层绑定即可。

## 实现代码

- https://github.com/load-more/vue-reactive-demo

## 参考资料

- https://www.cnblogs.com/canfoo/p/6891868.html
- https://www.bilibili.com/video/BV1Dr4y1c7xS

