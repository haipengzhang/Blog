---
title: iOS之Runloop
date: 2018-02-02 17:43:45
tags: iOS
---

Runloop,简而言之就是一个死循环用来接受内部和外部事件，定时任务等，和线程有密切的关系；

```
@autoreleasepool {
    NSLog(@"begin");
    int a = UIApplicationMain(argc, argv, nil, 	NSStringFromClass([AppDelegate class]));
    NSLog(@"end");
    return a;
}
```
上面的end永远也不会打印，UIApplicationMain会启动一个死循环；

```
function loop() {
    initialize();
    do {
		var message = get_next_message();
		process_message(message);
    } while (message != quit);
}

```

### Runloop的创建
苹果不允许直接创建Runloop，而是提供了两个方法：

* CFRunLoopGetMain()
* CFRunLoopGetCurrent()

```
CFRunLoopRef CFRunLoopGetMain() {
    return _CFRunLoopGet(pthread_main_thread_np());
}

CFRunLoopRef CFRunLoopGetCurrent() {
    return _CFRunLoopGet(pthread_self());
}
```
每个方法实际上都传入了一个**线程**参数，实际上是一个thread作为key，Runloop作为value的map。就是说一个线程对应一个最多只可能存在一个runloop的，非主线程不调用CFRunLoopGetCurrent是不会创建对应的Runloop。一般线程执行任务完毕的时候线程和对应Runloop都会被销毁，在某些需求下，需要对Runloop做一些特殊操作，实现线程保活；

### Runloop的API
RunLoop及其中的Mode的结构体：

```
struct __CFRunLoopMode {
    CFStringRef _name;            // Mode Name, 例如 @"kCFRunLoopDefaultMode"
    CFMutableSetRef _sources0;    // Set
    CFMutableSetRef _sources1;    // Set
    CFMutableArrayRef _observers; // Array
    CFMutableArrayRef _timers;    // Array
...
};

struct __CFRunLoop {
    CFMutableSetRef _commonModes;     // Set
    CFMutableSetRef _commonModeItems; // Set<Source/Observer/Timer>
    CFRunLoopModeRef _currentMode;    // Current Runloop Mode
    CFMutableSetRef _modes;           // Set
...
};
```
RunLoop中_modes保存所有的mode，commonModes保存的是标记为common的mode，每当RunLoop的内容发生变化时，RunLoop都会自动将_commonModeItems里的Source/Observer/Timer同步到具有“Common”标记的所有Mode里。基于此，可以通过把item加入_commonModeItems中来实现，即使currentMode切换了，commonModeItems里的任务一样不会被影响，子线程的timer就是应该这样处理。一个runloop的currentMode有且只会指定一种，通常是被系统指定，比如滑动的时候会切到UITrackingRunLoopMode；

{% asset_image RunLoop_0.jpg RunLoop_0 %}
一个RunLoop有很多个Mode,每个Mode里面有{CFRunLoopSourceRef}，[CFRunLoopTimerRef]，[CFRunLoopObserverRef]。每次往RunLoop里面添加任务的时候，只能为之指定一种mode。CFRunLoopSourceRef是基于source的事件来源，CFRunLoopTimerRef和NSTimer如出一辙，CFRunLoopObserverRef是一堆回掉函数指针的数组，用来RunLoop生命周期的回调；

Example:

```
//标记一个mode为commonmode
CFRunLoopAddCommonMode(CFRunLoopRef rl, CFRunLoopMode mode);	

//最后一个参数是modename，也可以是commonmodes字符串，会自动把item添加到_commonModeItems，然后同步到_commonModes。
CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFRunLoopMode mode); 

```

### Runloop的tips

* 当调用 performSelector:onThread: 时，实际上其会创建一个Timer 加到对应的线程去，同样的，如果对应线程没有RunLoop该方法也会失效；

* Timer没有加入到commonmode会出现不调用的情况；

* 基于Runloop obeserve，应用启动，休眠，消亡时刻对autoreleasepool的维护，一般逻辑代码（timer，响应事件）都是在对应线程的runloop中执行，通过obeserve时间点的控制能确保不会出现内存泄漏，还有UI刷新也和Runloop的obeserve相关，**开发者也可以创建obeserve实现在指定的时机执行一些操作**；

* runMode：beforeDate:，如果没有输入源或定时器连接到运行循环，则此方法立即退出；否则，重复调用在NSDefaultRunLoopMode中运行，直到指定的到期日期。**很多网络库底层为了stream open在子线程会schedule一个保活的custom_runloop（通过添加source），如果是主线程就不用这样了**；
