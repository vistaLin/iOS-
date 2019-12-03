#KVO
[TOC]

- KVO是如何实现的?
- 监察iOS属性变化还有其他方法吗?
- KVO的优劣势?
- 什么是KVC?
- KVO如何实现一对多?

- kvo是检测对象属性变化的观察者模式,可以

##问题

1.KVO是什么?
 - 键值监听,监测对象属性的改变
2.KVO是如何检测一对一
 - 通过注册观察者,对应的协议实现改变
3.使用KVO要注意什么?
 - 注意在delloc移除观察者,否则会崩溃
4.KVO的原理是什么?
 - [参考地址](https://juejin.im/post/5adab70cf265da0b736d37a8)


- delloc是什么?有什么作用?猜测一下它的实现

###1.KVO(Key-Value Observing)

- 键值监听,监测对象属性的改变,和NSNotification都是iOS的观察者模式,KVO是一对一以及一对多的观察

###4.KVO的原理

1.iOS用什么方式实现对一个对象的KVO?
 - 当一个对象使用KVO监听,iOS系统会修改这个对象的isa指针,改为指向一个全新的通过Runtime动态创建的子类,子类拥有自己的set方法实现,set方法实现内部会顺序调用**willChangeValueForKey方法、原来的setter方法实现、didChangeValueForKey方法,而didChangeValueForKey方法内部又会调用监听的observeValueForKeyPath:ofObject:change:context:监听方法**

2.如何手动触发
 - 自己调用**willChangeValueForKey和didChangeValueForKey**方法,即可在不改变属性值的情况下手动触发KVO,并且这两个方法缺一不可