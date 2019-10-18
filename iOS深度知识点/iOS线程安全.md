#iOS线程安全

[TOC]

- 多线程操作共享数据不会出现想不到的结果就是线程安全的,否则,是线程不安全的。
- 单线程是不会出现线程安全问题,因为单线程代码的执行顺序是可以确定的。
 - 异步在新的线程中执行任务,相当于开启了新线程

##如何解决线程安全问题?

1.将多线程直接修改成单线程.
2.线程不安全是由多线程访问和修改共享资源而引起不可预测的结果,因此,访问共享资源而不修改共享资源也可以保证线程安全,比如:设置只读属性的全局变量。
3.使用锁，在读取共享资源的时候不能修改。

##锁的种类

- [可阅读参考连接](不再安全的 OSSpinLock：https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)

###1.自旋锁OSSpinLock

- 自旋锁的原理就是死循环。当a线程获得锁的以后,b线程就会一直等待a线程解锁。b线程会忙等会消耗大量的CPU的时间,所以,自旋锁用在临界值区执行时间比较短的环境性能会很高.

- demo
```OC
#import OSSpinLock lock = OS_SPINLOCK_INIT;
OSSpinLockLock(&lock);
//需要执行的代码
OSSpinLockUnlock(&lock);
//OSSPINLOCK_DEPRECATED_REPLACE_WITH(os_unfair_lock)
//苹果在OSSpinLock注释表示被废弃，改用不安全的锁替代
```

###2.dispatch_semaphore:

- dispatch_semaphore和自旋锁不一样,首先会先将信号量减一,并判断是否大于等于0,如果是,则返回0,并继续执行后续代码,否则,使线程进入睡眠状态,让出cpu时间,直到信号量大于0或者超时,则线程会被重新唤醒执行后续操作。

```OC
dispatch_semaphore_t lock = dispatch_semaphore_create(1);    //传入的参数必须大于或者等于0，否则会返回Null
long wait = dispatch_semaphore_wait(lock, DISPATCH_TIME_FOREVER);    //wait = 0，则表示不需要等待，直接执行后续代码；wait != 0，则表示需要等待信号或者超时，才能继续执行后续代码。lock信号量减一，判断是否大于0，如果大于0则继续执行后续代码；lock信号量减一少于或者等于0，则等待信号量或者超时。
//需要执行的代码
long signal = dispatch_semaphore_signal(lock);    //signal = 0，则表示没有线程需要其处理的信号量，换句话说，没有需要唤醒的线程；signal != 0，则表示有一个或者多个线程需要唤醒，则唤醒一个线程。（如果线程有优先级，则唤醒优先级最高的线程，否则，随机唤醒一个线程。）
```

###3.pthread_mutex:

- pthread_mutex表示互斥锁,和信号量的实现原理类似,也是阻塞线程并进入睡眠,需要进行上下文切换。

```OC
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_NORMAL);
    
pthread_mutex_t lock;
pthread_mutex_init(&lock, &attr);    //设置属性
    
pthread_mutex_lock(&lock);    //上锁
//需要执行的代码
pthread_mutex_unlock(&lock);    //解锁
```

###4.NSLock:

- NSLock在内部封装了一个pthread_mutex,属性为PTHREAD_MUTEX_ERRORCHECK

```OC
NSLock *lock = [NSLock new];
[lock lock];
//需要执行的代码
[lock unlock];
```

###5.NSCondition:

- NSCondition封装了一个互斥锁和条件变量。互斥锁保证线程安全，条件变量保证执行顺序。

```OC
NSCondition *lock = [NSCondition new];
[lock lock];
//需要执行的代码
[lock unlock];
```

###6.pthread_mutex(recursive):

- pthread_mutex锁的一种,属于递归锁,一般一个线程只能申请一把锁,但是,如果是递归锁,则可以申请多把锁,只要上锁和解锁的操作数量就不会报错。

```OC
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
    
pthread_mutex_t lock;
pthread_mutex_init(&lock, &attr);    //设置属性
    
pthread_mutex_lock(&lock);    //上锁
//需要执行的代码
pthread_mutex_unlock(&lock);    //解锁
```

###7.NSRecursiveLock:

- 递归锁,pthread_mutex(recusive)的封装。

```OC
NSRecursiveLock *lock = [NSRecursiveLock new];
[lock lock];
//需要执行的代码
[lock unlock];
```

###8 NSConditionLock:

- NSConditionLock借助NSCondition来实现,本质是生产者-消费者模型

```OC
NSConditionLock *lock = [NSConditionLock new];
[lock lock];
//需要执行的代码
[lock unlock];
```

###9 @synchronized:

- 一个对象层面的锁,锁住了整个对象,底层使用了互斥递归锁来实现

```OC
NSObject *object = [NSObject new];
@synchronized(object) {
  //需要执行的代码
}
````

