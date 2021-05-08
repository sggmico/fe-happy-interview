本文将带你手写new运算符的实现。记得自己动手敲一敲、讲一讲，加深理解和记忆~

> new运算符用来创建一个用户定义的对象类型的实例或具有构造函数的内置对象的实例。—— [MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/new)

## 操作分析

下面我们分析下，当构造函数使用new运算符实例化创建对象时，默认（隐式）地执行了哪些操作：

1. 创建新对象： obj。不同创建方式如下：
   1. 对象字面量：obj = {} 
   2. obj = new Object()
   3. obj = Obejct.create() 。`性能更好，推荐`
2. 构建obj的原型：将obj的原型指向构造函数的原型对象，即：obj.\_\_proto\_\_ = Constructor.prototype `（该方式不推荐）` 。 [不推荐说明 —— MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/proto)
3. 绑定this到obj：构造函数绑定this到obj，并执行。可实现obj能访问到构造函数内属性和方法
4. 返回值判断：构造函数内有返回值res且res为对象类型，直接返回res，否则返回新对象obj

## **简单实现：**

```js
function new_operator(_constructor, ...args) {
  // ① 创建新对象
  
  let obj = {}
  
  // ② 构建obj的原型
  
  obj.__proto__ = _constructor.prototype
  
  // ③ 绑定this到obj
  
  let res = _constructor.apply(obj, args)
  
  // ④ 返回值判断。（注：res值为对象，即 Object, Array, Function 类型，才可被返回）
  
  return res instanceof Object ? res : obj
}
```

以上实现，在构建新对象obj的原型引用时，直接操作obj上的 `__proto__`。会存在以下问题：

1. `__proto__` 属性已经在最新的Web标准中删除，考虑操作安全，不推荐使用。如果想直接操作原型引用，推荐使用 [`Object.getPrototypeOf`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/getPrototypeOf)/[`Reflect.getPrototypeOf`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Reflect/getPrototypeOf)  和[`Object.setPrototypeOf`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/setPrototypeOf)/[`Reflect.setPrototypeOf`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Reflect/setPrototypeOf)
2. 1中两种方式操作对象的原型`[[Prototype]]`都是一个缓慢的操作。考虑性能，应避免。推荐使用`Ojbect.create()` 创建一个新的且可以继承`[[Prototype]]`的对象。

## **优化实现：**

```js
function new_operator(_constructor, ...args) {
	// ① 创建新对象obj，并关联obj原型到构造函数原型对象上
	
  let obj = Object.create(_constructor.prototype)
  
  // ② 执行构造函数，且绑定this到新对象Obj上，实现继承。同时接受返回值res
  
  let res = _constructor.apply(obj, args)
  
  // ③ 返回值判断
  
  return res instanceof Object ? res : obj
}
```
本文有任何纰漏，欢迎评论区指正。🤝

本文首发笔者github「原文链接」，其他平台（知乎、掘进）同步更新，未经许可禁止转载！

希望本篇小文能对前端小伙伴有所帮助！ 如果你是那一位，记得用小手指轻轻地留下`点赞`和`关注`吧！让笔者知道，因为这份动力弥足珍贵！

![](https://tva1.sinaimg.cn/large/0081Kckwly1gkl8zxkodij30go0go436.jpg)



## 也许这些你也需要

1. [【面试题】请实现一个”递增计时器“](https://mp.weixin.qq.com/s/N_at9DMwmqudca4syuU87A)
2. [【性能优化】0202年了，函数节流与防抖还不用起来？](https://mp.weixin.qq.com/s/gNz2F-vYUqKccd3_tS6-yA)
3. [设计模式 —— 简单工厂模式](https://mp.weixin.qq.com/s/kyyzKw8hUVMllmmoUI79nQ)
4. [Linux免密登录远程主机服务器](https://mp.weixin.qq.com/s/qCFzKaGnXKVsFv7lUkoKYg)



👇 直接点击「原文地址」，享用更多前端知识干货的分享和讨论，开拓前端视野，强势进阶！欢迎 `start` 或 `follow`。



See you next time ~~