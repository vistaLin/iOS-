#Runtime

- 学习到:数据结构的Class

- RunTime就是消息机制
- [参考地址](https://juejin.im/post/5a37562451882506e50cbc9a)

- 如何学习RunTime?
1.查看iOS开发高手课推荐的资料
2.网络搜索
3.查询面试资料说的
4.查看书籍

1.RunTime的应用
2.RunRime消息转发的流程
2.RunTime实现原理

- 成员变量的基础应用 学习成员变量的属性和本质

##问题

- 标题class:isa指针,对象可以通过isa指针找它的类,类需要通过isa找到它的元类,那么isa指针是因为一个地址来取寻址的吗?指针是地址吗?它是如何寻址的?
- 标题class:methodLists是一个指针的指针,指针的指针该如何理解?为何说修改该指针指向指针的值,就可以添加方法,为为什么变量指针不可以添加呢?是因为值固定还是因为什么?
- 标题元类:既然我们向一个对象发送消息时,runtime会在这个对象所属的这个类的方法列表中查找方法,那么查找不到又是如何去查找父类的方法呢?
- SEL的实质如何hash话,hash算法吗?
- c语言的结构体,指针,函数指针,指向指针的指针,仔细研究一下
- 理解8Cache结构体的含义

##一、RunTime简介

- 对于C语言,函数的调用在编译的时候就会决定调用哪个函数。对于OC的方法,属于**动态调用过程**,在编译的时候并不能决定真正调用哪个函数,只有在真正运行的时候才会根据函数的名称找对应的函数来调用。
- 在Runtime中,对象可以用C语言中的结构体表示,而方法可以用C函数来实现,另外在加上了一些额外的特性。这些结构体和函数被Runtime函数封装后,让OC的面向对象编程变为可能。

##二、Objective-C中的数据结构


###1.id

- **通过isa指针找到这个实例对象所属的类,然后在其类和父类方在列表中找到selector指向的方法**
- 运行期系统不知道某个对象的类型,对象类型在并不是在编译器就知道了,而是在运行期查找,OC特殊类型id可以表示任意对象类型。每个对象结构体的首个成员是Class类的变量,该变量定义了对象所属的类,通常称为isa指针。
```OC
struct objc_object {
   Class isa;
} *id;
```
####objc_object

- objc_object是表示一个类实例的结构体,它的定义如下(objc/objc.h):

```OC
struct objc_object {
   Class isa OBJC_ISA_AVAILABILITY;
}
typedef struct objc_object *id;
```

- 上面这个结构体只有一个字体,即指向其类的isa指针。这样，当我们向一个Objective-C对象发送消息时，运行时库会根据实例对象的isa指针找到这个实例对象所属的类。Runtime库会在类的方法类表及父类的方法列表中去寻找与消息对应的selector只想的方法,找到后即运行这个方法

###2.Class

- Objective-C中,类是由Class类型来表示,它实际上是一个指向objc_class结构体的指针

```OC
typedef struct objc_class *Class;

struct objc_class {
  Class isa OBJC_ISA_AVAILABILITY;
#if !_OBJC2_
  Class super_class OBJC2_UNAVAILABLE; // 父类
  const char *name OBJC2_UNAVAILABLE; // 类名
  long version OBJC2_UNAVAILABLE; // 类的版本信息,默认为0
  long info  OBJC2_UNAVAILABLE; // 类信息,供运行期使用的一些标识
  long instance_size  OBJC2_UNAVAILABLE; // 该类的实例变量大小
  struct objc_ivar_list *ivars OBJC2_UNAVAILABLE; // 该类的成员变量链表
  struct objc_method_list **methodLists OBJC2_UNAVAILABLE; //方法定义的链表
  struct objc_cache *cache OBJC2_UNAVAILABLE; // 方法缓存
  struct objc_protocol_list *protocols  OBJC2_UNAVAILABLE; // 协议链表
#endif
}
```

Class的结构体重的几个主要变量:

1.isa:结构体的首个变量也是isa指针,这说明Class本身也是Objective-C中的对象。对象需要通过isa指针找到它的类,类需要通过isa找到它的元类,这在调用实例方法和类方法的时候起到重要的作用.
2.super_class:结构体里还有个变量是super_class,它定义了本类的超类。类对象所属类型(isa指针所指向的类型)是另一个类,叫做**元类**.
3.ivars:成员变量列表,类的成员变量都在ivars里面.
4.methodLists:方法列表,类的实例方法都在methodLists里,类方法在元类的methodLists里面.methodLists是一个指针的指针,通过修改该指针指向指针的值,就可以动态的为某一个类添加成员方法.这也就是Category实现的原理,同时也说明了Category只可以为对象添加成员方法,不能添加成员变量.
5.cache:方法缓存列表,objc_msgSend每调用一次方法后,就会把该方法缓存在cache列表中,下次调用的时候,会优先从cache列表中寻找,如果cache没有,才从methodLists中查找方法.提高效率.

####元类(Meta Class)

- meta-class是一个类对象的类。所有的类自身也是一个对象，我们可以向这个对象发送消息（即调用类方法）。既然是对象，那么它也是一个objc_object指针,它包含一个指向其类的一个isa指针。这个类的isa指针必须指向一个包含这些类方法的一个objc_class结构体.这就引出了meta-class概念,meta-class中存储这一个类的所有类方法.所以,调用类方法的这个类对象的isa指针指向的就是meta-class,当我们向一个对象发送消息时,runtime会在这个对象所属的这个类的方法列表中查找方法;而向一个类发送消息时,会在这个类的meta-class的方法列表中查找.
- meta-class也是一个类,也可以向它发送一个下去,OC设计这让所有的meta-class的isa指向积累的meta-lcass,以此作为它们所属类.
- 实例对象的isa指向类,类对象的isa指向元类,元类对象isa指针指向一个"根元类".
> 1.Class是一个指向objc_class结构体的指针,而id是一个指向objc_object结构体的指针,其中的isa是一个指向objc_class结构体的指针.其中的id就是我们所说的对象,Class就是我们所说的类
> 2.isa指针不总是指向实例对象所属的类,不能依靠它来确定类型,而是应该用isKindOfClass:方法来确定实例对象的类.因为KVO的实现机制就是将被观察对象的isa指针指向一个中间类而不是真实的类

####Category

-Category是表示一个指向分类的结构体的指针,定义如下:
```OC
typedef struct objc_category *Category
struct objc_category {
    char *category_name   OBJC2_UNAVAILABLE; // 分类名
    char *class_name  OBJC2_UNAVAILABLE; // 分类所属的类名
    struct objc_method_list *instance_methods OBJC2_UNAVAILABLE; // 实例方法列表
    struct objc_method_list *class_methods OBJC2_UNAVAILABLE; // 类方法列表
    struct objc_protocol_list *protocols OBJC2_UNAVAILABLE; // 分类所实现的协议列表
}
```

- 这个结构体主要包含了分类定义的实例方法和类方法,其中instance_methods列表是objc_class中方法列表的一个子集,而class_methods列表是元类方法列表的一个子集.可发现,类别中没有ivar成员变量指针,也就意味着:**类别中不能够添加实例变量和属性**

```OC
struct objc_ivar_list *ivars OBJC2_UNAVAILABLE; // 该类的成员变量链表
```

###3.SEL

- SEL是选择子的类型,选择子指的就是方法的名字.定义:

```OC
typedef struct objc_selector *SEL;
```

- SEL实际上就是根据方法名hash化了一个字符串,而对于字符串的比较仅仅需要比较他们的地址就可以了.找到方法的地址,进而调用即可

###4.Method

Method代表类中的某个方法的类型,在Runtime的头文件中的定义如下:

```OC
typedef struct objc_method *Method
```

objc_method的结构体定义如下:

```OC
struct objc_method {
  SEL method_name OBJC2_UNAVAILABLE; // 方法名
  char *method_types OBJC2_UNAVAILABLE; // 方法类型
  IMP method_imp  OBJC2_UNAVAILABLE; // 方法实现
}
```

- 1.method_name: 方法名
- 2.method_types:方法类型,主要存储着方法的参数类型和返回值类型.
- 3.IMP:方法的实现,函数指针.**class_copyMethodList(Class cls, unsigned int *outCout)**可以使用这个方法获取某个类的成员方法列表

Method用于表示类定义中的方法,该结构体中包含一个SELh和IMP,相当于在SEL和IMP之间作了一个映射,有了SEL,就可以找到对应IMP,从而调用方法的实现代码

###5.Ivar

Ivar代表类中实例变量的类型,在Runtime的头文件中的定义如下:
```OC
typedef struct objc_ivar *Ivar;
```
objc_ivar的定义如下:
```OC
struct objc_ivar {
   char *ivar_name OBJC2_UNAVAILABLE;
   char *ivar-type OBJC2_UNAVAILABLE;
   int ivar_offset OBJC2_UNAVAILABLE;
#ifdef __LP64__
  int space OBJC2_UNAVAILABLE;
#endif
}
```

**class_copyIvarList(Class cls, unsigned int *outCout)**可以使用这个方法获取某个类的成员变量列表.

###6.objc_property_t

objc_property_t是属性,在Runtime的头文件中的定义如下:
```OC
typedef struct objc_property *objc_property_t;
```

**class_copyPropertyList(Class cls, unsigned int *outCount)**可以使用这个方法获取某个属性列表.

###7.IMP

IMP在Runtime的头文件中的定义如下:
```OC
typedef id (*IMP)(id, SEL, ...);
```

IMP是一个函数指针,指向方法实现的地址, 它是由编译器生成的,定义如下:
```OC
id (*IMP)(id,SEL, ...)
```
第一个参数:是只想self的指针(如果是实例方法,则是类实例的内存地址;如果是类方法,则是指向原来的指针)
第二个参数:是方法选择器(selector),接下来的参数:方法的参数列表


###8.Cache

Cache在Runtime头文件中定义如下:
```OC
typedef struct objc_cache *Cache
```

objc_cache的定义如下:
```OC
struct objc_cache {
   unsigned int mask OBJC2_UNAVAILABLE;
   unsigned int occupied OBJC2_UNAVAILABLE;
   Method buckets[1] OBJC2_UNAVAILABLE;
}
```

每调用一次方法后,不会直接在isa指向的类的方法列表(methodLists)中遍历查找能够响应消息的方法,因为这样效率太低。它会把该方法缓存到cache列表中,下次的时候,就直接优先从cache列表中寻找,如果cache没有,才从isa指向的类的方法列表(methodLists)中查找方法.提高效率

##三.发送消息(objc_msgSend)

OC调用方法就是"消息传递", 消息有"名称"(name)或者"选择子"(selector), 也可以接受参数,可能还有返回值.对象收到消息之后,具体调用哪个方法由运行期解决.所以oc称为动态语言

```OC
id returnValue = [someObject message:parm];
```

someObject叫做"接收者"(receiver),message是"选择子"(selector),选择子和参数结合起来就叫做"消息"(message).编译器看到消息后,将其转换为C语言函数调用,所调用的函数乃是消息传递机制中的核心函数**objc_msgSend**,原型如下:

```OC
id objc_msgSend(id select, SEL_cmd, ...);
```
第一个参数是接受者(receiver), 第二个参数是选择子(selector),后续参数就是消息中心传递的那些参数(parm),其顺序不变.
编译器会把上面的那个消息转换成为:
```OC
id returnValue objc_msgSend(someObject, @selector(message:),parm);
```

**objc_msgSend**发送消息的原理:
1.检测这个selector是不会要被忽略的.
> respondsToSelector 检测是否实现 如果没有实现自然就是忽略

2.检测这个target对象是不是nil对象.
3.根据target(objc_object)对象的isa指针获取它所对应的类(obj_class)
4.查看缓存中是否存在方法,系统把近期发送过的消息记录其中,优先在类(class)的cache里面查找是否有与选择子(selector)名称相符的方法.如果有,则找到objc_method中的IMP类型(函数指针)的成员method_imp去找到实现内容,并执行;如果缓存中没有,那么到该类的方法表(methodLists)查找该方法,依次从后往前查
5.如果没有在类(class)找到,再到父类(super_class)查找,直到根类.
6.一旦找到与选择子(selector)名称相符的方法,就跳至其实现代码.
7.如果没有找到,就会执行消息转发(message forwarding)的第一步动态解析

**如果是调用类方法**objc_class的isa指向该类的元类(metaClass)如果调用类方法的话,那么就会利用objc_class中的成员isa找到元类(metaclass),然后寻找方法,直至根metaclass,没有找到的话仍然进入动态解析.

##四.消息转发(message forwarding)

当一个对象能接收一个消息时,就会走正常的方法调用流程.但如果一个对象无法接收指定消息时.又会发生什么事呢?默认情况下,如果是以**[object message]**的方式调用方法,如果**object**无法响应**message**消息时,编译器会报错.但如果是以**perform ...**的形式调用,则需要等到运行时才能确定**object**是否能接受**message**消息.如果不能,则程序崩溃

当我们不能确定一个对象是否能接收某个消息时,会先调用respondsToSelector:来判断一下.如下代码:

```OC
if([self respondsToSelector:@selector(method)]){
    [self performSelector:@selector(method)];
}

此方法如何应用??用在什么地方??
```

当我们不使用**respondsToSelector:**判断的情况。当一个对象无法接收某一个消息时，就会启动所谓**消息转发(message forwaarding)**机制,通过这一机制,可以告诉对象如何处理未知的消息。默认情况下，对象接收到未知的消息，会导致程序崩溃，异常信息由NSObject的**doesNotRecongnizeSelector**方法抛出。消息转发机制分为三个步骤：
1.动态方法解析
2.备用接收者
3.完整转发

###动态方法解析

对象在接收到位置的消息时,首先会调用所属类的类方法**+resolveInstanceMethod:**(实例方法)或者**+resolveClassMethod:**(类方法)。在这个方法中，我们有机会为该未知消息新增一个**处理方法**。不过使用该方法的前提是我们已经实现了该**处理方法**,只需要在运行时通过**class_addMethod**函数动态添加到类里面就可以了.如下代码:

```OC
void functionForMethod1(id self, SEL_cmd) {
   NSLog()
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
   NSString *selectorString = NSStringFromSelector(sel);
   if([selectorString isEqualToString:@"method1"]) {
      class_addMethod(self.class, @selector(method1), (IMP)functionForMethod1, "@:");
   }
   return [super resolveInstanceMethod:sel];
}
```

```OC
void otherEat(id self, SEL cmd) {
    NSLog(@"blog.yoonangel.com");
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if ([NSStringFromSelector(sel) isEqualToString:@"eat"]) {
        class_addMethod(self, sel, (IMP)otherEat, "v@");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}

```

class_addMethod方法可谓是核心，那么依次来看他的参数的含义：

first：添加到哪个类
second：添加方法的方法编号（选择子）
third：添加方法的函数实现(IMP函数指针)
fourth：IMP指针指向的函数返回值和参数类型
v代表无返回值void @代表id类型对象->self  :代表选择子SEL->_cmd

"v@:" v代表无返回值void，如果是i则代表int 无参数
"i@:" 代表返回值是int类型，无参数
"v@:i@:" 代表返回值是void类型，参数是int类型，存在一个参数（多参数依次累加）"v@:@@" 代表 两个参数的没有返回值。

这种方案更多的是为了实现@dynamic属性












