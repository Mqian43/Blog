## 手写vue

vue是采用数据劫持的方式配合发布-订阅模式实现MVVM的响应式，通过**Object.defineProperty()**来劫持各个属性的getter & setter,在数据发生变化时候，发布消息给依赖收集器，去通知观察者，做出对应的回调函数，更新视图

MVVM作为绑定的入口，整合Observer,Compile和Watcher三者，通过Observer来监听model数据的变化，通过Compile来解析编译模板指令，最终利用Watcher搭起Observer,Compile之间的通信桥梁，达到数据变化=>视图更新；视图变化 => 数据mode变更的双向绑定的效果

##### 目录结构

![image-20210104135857233](C:\Users\qm\AppData\Roaming\Typora\typora-user-images\image-20210104135857233.png)

##### index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
</head>
<body>
  <div id="app">
    <h3>{{msg}}</h3>
    <h3>{{person.name}} -- {{person.age}}</h3>
    <div></div>
    <div v-text="msg"></div>
    <div v-html="msg"></div>
    <input type="text" v-model="person.name">
    <button v-on:click="handle">打印this</button>
    <button @click="handle">打印this</button>
    <img style="width: 100px;height: 100px;" v-bind:src="baiduImgUrl" alt="">
  </div>
</body>
<!-- <script src="./vue.js"></script> -->
<script src="./Observer.js"></script>
<script src="./MVue.js"></script>
<script>
let vm = new Vue({
  el:'#app',
  data:{
    person:{
      name:'小明',
      age: 18,
    },
    msg: '手写vue',
    baiduImgUrl: 'https://ss2.bdstatic.com/70cFvnSh_Q1YnxGkpoWK1HF6hhy/it/u=2843712642,3660282042&fm=26&gp=0.jpg'
  },
  methods: {
    handle () {
      console.log(this)
      this.person.name = '学习vue'
      this.msg = '1221313'
    } 
  },
})
</script>
</html>
```



##### MVue.js

```js
const compileUtil = {
  getVal(expr, vm) {
    return expr.split('.').reduce((data, currentVal) => {
      return data[currentVal]
    }, vm.$data)
  },
  setVal(expr, vm, inputVal) {
    expr.split('.').reduce((data, currentVal, index, arr) => {
      if (index !== arr.length-1) {
        return data[currentVal]
      } else {
         data[currentVal] = inputVal
      }
    }, vm.$data)
  },
  getContentVal(expr, vm){
    return expr.replace(/\{\{(.+?)\}\}/g, (...args) => {
      return this.getVal(args[1], vm)
    })
  },
  text(node, expr, vm) {
    let value
    if (expr.indexOf('{{') !== -1) {
      value = expr.replace(/\{\{(.+?)\}\}/g, (...args) => {
        new Watcher(vm, args[1], () => {
          this.updater.textUpdater(node, this.getContentVal(expr, vm))
        })
        return this.getVal(args[1], vm)
      })
    } else {
      new Watcher(vm, expr, (newVal) => {
        this.updater.textUpdater(node, newVal)
      })
      value = this.getVal(expr, vm)
    }
    this.updater.textUpdater(node, value)
  },
  html(node, expr, vm) {
    const value = this.getVal(expr, vm)
    new Watcher(vm, expr, (newVal) => {
      this.updater.htmlUpdater(node, newVal)
    })
    this.updater.htmlUpdater(node, value)
  },
  model(node, expr, vm) {
    const value = this.getVal(expr, vm)
    // 绑定更新函数  数据 =》 视图
    new Watcher(vm, expr, (newVal) => {
      this.updater.modelUpdater(node, newVal)
    })
    // 视图 => 数据 => 视图  
    node.addEventListener('input', (e) => {
      // 设置值
      this.setVal(expr, vm, e.target.value)
      // this.updater.modelUpdater(node, newVal)
    })
    this.updater.modelUpdater(node, value)
  },
  on(node, expr, vm, eventName) {
    let fn = vm.$options.methods && vm.$options.methods[expr]
    node.addEventListener(eventName, fn.bind(vm), false)
  },
  bind(node, expr, vm, eventName) {
    const value = this.getVal(expr, vm)
    node[eventName] = value
  },
  updater: {
    textUpdater(node, value) {
      node.textContent = value
    },
    htmlUpdater(node, value) {
      node.innerHTML = value
    },
    modelUpdater(node, value) {
      node.value = value
    },
  },
}

/**
 * 编译类
 * 将el放入文档碎片  编译完成后重新渲染
 */
class Compile {
  constructor(el, vm) {
    this.el = this.isElementNode(el) ? el : document.querySelector(el)
    this.vm = vm
    // 1.获取文档对象碎片  放入内存中  减少页面的回流和重绘
    const fragment = this.node2Fragment(this.el)
    // 2.编译模板
    this.compile(fragment)
    // 3.追加子元素到根元素
    this.el.appendChild(fragment)
  }
  isElementNode(node) {
    return node.nodeType === 1
  }
  node2Fragment(el) {
    const f = document.createDocumentFragment()
    let firstChild
    while ((firstChild = el.firstChild)) {
      f.appendChild(firstChild)
    }
    return f
  }
  compile(fragment) {
    // 1.获取子节点
    const nodes = fragment.childNodes
    let nodeArr = [...nodes]
    nodeArr.forEach((child) => {
      if (this.isElementNode(child)) {
        // console.log('是元素节点', child)
        this.compileElement(child)
        if (child.childNodes && child.childNodes.length) {
          this.compile(child)
        }
      } else {
        this.compileText(child)
      }
    })
  }
  compileElement(node) {
    const attrs = [...node.attributes]
    attrs.forEach((attr) => {
      const { name, value } = attr
      if (this.isDirective(name)) {
        const [, directive] = name.split('-')
        const [dirName, eventName] = directive.split(':')
        compileUtil[dirName](node, value, this.vm, eventName)
        // 删除元素上的指令标签
        node.removeAttribute('v-' + directive)
      } else if (this.isEventName(name)) {
        let [, eventName] = name.split('@')
        compileUtil['on'](node, value, this.vm, eventName)
      }
    })
  }
  compileText(node) {
    const content = node.textContent
    if (/\{\{(.+?)\}\}/.test(content)) {
      compileUtil['text'](node, content, this.vm)
    }
  }
  isDirective(attrName) {
    return attrName.startsWith('v-')
  }
  isEventName(attrName) {
    return attrName.startsWith('@')
  }
}
// vue 类
class Vue {
  constructor(options) {
    this.$el = options.el
    this.$data = options.data
    this.$options = options
    if (this.$el) {
      // 1.实现一个数据观察者
      new Observer(this.$data)
      // 2.实现一个指令解析器
      new Compile(this.$el, this)
      this.proxyData(this.$data)
    }
  }
  proxyData(data) {
    for (const key in data) {
      Object.defineProperty(this, key, {
        get() {
          return data[key]
        },
        set(newVal) {
          data[key] = newVal
        },
      })
    }
  }
}

```



##### Observer.js

```js
class Watcher {
  constructor(vm, expr, cb) {
    this.vm = vm
    this.expr = expr
    this.cb = cb
    this.oldVal = this.getOldVal()
  }
  update() {
    const newVal = compileUtil.getVal(this.expr, this.vm)
    if (newVal !== this.oldVal) {
      this.cb(newVal)
    }
  }
  getOldVal() {
    Dep.target = this
    const oldVal = compileUtil.getVal(this.expr, this.vm)
    Dep.target = null
    return oldVal
  }
}

class Dep {
  constructor() {
    this.subs = []
  }
  // 收集观察者
  addSub(watcher) {
    this.subs.push(watcher)
  }
  // 通知观察者去更新
  notify() {
    this.subs.forEach((w) => w.update())
  }
}

class Observer {
  constructor(data) {
    this.observe(data)
  }
  observe(data) {
    if (data && typeof data === 'object') {
      Object.keys(data).forEach((key) => {
        this.defineReactive(data, key, data[key])
      })
    }
  }
  defineReactive(data, key, value) {
    // 递归遍历
    this.observe(value)
    const dep = new Dep()
    Object.defineProperty(data, key, {
      enumerable: true,
      configurable: false,
      get() {
        // 订阅数据变化时， 往Dep中添加观察者
        Dep.target && dep.addSub(Dep.target)
        return value
      },
      // 使用箭头函数， 避免this指向问题
      set: (newVal) => {
        this.observe(newVal)
        if (newVal !== value) {
          value = newVal
        }
        // 告诉Dep 通知变化
        dep.notify()
      },
    })
  }
}

```

