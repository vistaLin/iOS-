#iOS查询知识点

[TOC]

##中级

###category和extension的区别

- extension在编译期决定的,是类的一部分,可以添加实例变量。而category是在运行期决议的,无法添加实例变量。
> 因为在运行期,对象的内存布局已经确定,如果添加实例变量就会破坏类的内部布局,这对变异型语言是灾难性的
> [参考地址](https://tech.meituan.com/2015/03/03/diveintocategory.html)
> extension如何实现的?如何动态添加实例变量的?  感觉就是给类添加一个私有属性,只能在内部使用的感觉
> category如何实现的?它动态添加方法吗?它的本质是什么? 
> 实例变量和属性变量什么区别?为什么category的可为（可以添加实例方法，类方法，甚至可以实现协议，添加属性）和不可为（无法添加实例变量）
> 需要写一遍关系category和extension的
> [runtime](https://juejin.im/post/5a37562451882506e50cbc9a#heading-15)
> rumtime说Category没有ivars成员变量列表,而美团的文章说Category的结构体
##高级

###Runtime

- 详情参考文档Runtime.md文档
- runtime实际上就是OC的消息机制

1.objc_msgSend发送消息
2.没有找到消息即执行消息转发(message forwarding)

####objc_msgSend发送顺序

1.检测selector是不是要被忽略
2.检测这个target对象是不是nil对象
3.首先会根据target对象的isa指针获取它对应的类
4.先查看缓存中是否存在方法
5.没有在类里找到,再到父类直至根类
6.如果找到selector名称相符的方法,就跳至实现
7.如果没有找到,就执行消息转发(message forwarding)的动态解析
