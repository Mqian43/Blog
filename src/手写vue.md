## 手写vue

vue是采用数据劫持的方式配合发布-订阅模式实现MVVM的响应式，通过**Object.defineProperty()**来劫持各个属性的getter & setter,在数据发生变化时候，发布消息给依赖收集器，去通知观察者，做出对应的回调函数，更新视图

MVVM作为绑定的入口，整合Observer,Compile和Watcher三者，通过Observer来监听model数据的变化，通过Compile来解析编译模板指令，最终利用Watcher搭起Observer,Compile之间的通信桥梁，达到数据变化=>视图更新；视图变化 => 数据mode变更的双向绑定的效果



