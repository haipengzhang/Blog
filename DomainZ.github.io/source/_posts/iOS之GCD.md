---
title: iOS之GCD
date: 2019-08-30 15:24:16
tags:
---

### 什么是GCD
GCD（Grand Central Dispatch）是异步执行任务的技术之一（iOS中其他多线程技术：pthread、NSThread、NSOperation）。开发者只需要定义想要执行的任务并追加到适当的Dispatch Quue中，GCD就能生成必要的线程并执行，**通过GCD提供的系统级线程管理可以提高执行效率**。

#### GCD队列
---
**队列与线程的关系**

队列和线程并非拥有关系，队列是任务容器（一种数据结构），CPU从队列中取出任务，放到对应的线程上去执行。

**串行队列与并发队列**

* 串行队列同时执行的处理数只有一个，按照顺序执行。
* 并发队列执的行顺序会取决于处理的任务量和系统的状态（CPU核数、CPU负荷等）。
* 多个串行队列可并发执行，每个串行队列都使用各自的一个线程。

当生成多个串行队列时，各个串行队列将并发执行。一旦生成串行队列并追加任务处理，**系统对于一个串行队列就只使用一个线程。**如果使用过多线程，就会消耗大量内存，引起大量的上下文切换，大幅度降低系统的响应性能。**并发队列不会出现以上问题，不管生成多少，XNU内核只使用有效管理的线程。**

**主队列与全局队列**

* 追加到主队列的任务在主线程的RunLoop中执行，如更新用户界面的处理必须追加到主队列中，与NSObject类的performSelectorOnMainThread实例方法相同。

* 全局队列无需逐个创建并发队列，只要获取使用即可。全局队列有四个优先级，优先级只是大致判断，并不能保证线程的实时性。

#### GCD函数

**1.Dispatch\_set\_ target_queue**

变更队列优先级

```
/**
 * param1 要变更的队列，不能指定主队列和全局队列
 * param2 目标队列，指定全局队列
 */
dispatch_set_target_queue(myQueue, backgroundQueue);
```

防止多个串行队列并发执行

```
- (void)dispatch_queue_test_2 {
    NSMutableArray *array = [NSMutableArray array];
    // 设置目标队列
    dispatch_queue_t targetQueue = dispatch_queue_create("com.target.serialQueue", DISPATCH_QUEUE_SERIAL);
    for (int i = 0; i < 5; i ++) {
        dispatch_queue_t serialQueue = dispatch_queue_create("com.example.serialQueue", DISPATCH_QUEUE_SERIAL);
        // 给每个串行队列指定相同的目标队列
        dispatch_set_target_queue(serialQueue, targetQueue);
        [array addObject:serialQueue];
    }
    [array enumerateObjectsUsingBlock:^(dispatch_queue_t queue, NSUInteger idx, BOOL * _Nonnull stop) {

        dispatch_async(queue, ^{
            NSLog(@"执行队列：%ld",idx);
        });
    }];
}
```

输出

```
执行队列：0
执行队列：1
执行队列：2
执行队列：3
执行队列：4

```

**2.Dispatch_after**

dispatch_after函数是在**指定时间追加任务到指定队列中**，并不是在指定时间后执行任务。想大致延迟任务时，该函数非常有效。

**3.Dispatch Group**

Dispatch Group适用于多个任务执行结束后，再执行某个指定的任务。创建任务组使用dispatch_group_create函数，追加任务使用dispatch_group_async函数。

```
- (void)dispatch_notify {
    // 组
    dispatch_group_t group = dispatch_group_create();

    // 队列
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

    // 5个任务
    for (NSInteger index = 0; index < 5; index ++) {

        /**
         * param1 组
         * param2 队列
         */
        dispatch_group_async(group, queue, ^{

            NSLog(@"任务%ld", index);
        });
    }

    /**
     * 监听任务的完成
     * param1 组
     * param2 队列
     */
    dispatch_group_notify(group, queue, ^{
        NSLog(@"任务完成");
    });
}
```

输出：

```
任务1
任务2
任务0
任务3
任务完成

```

dispatch\_group\_wait函数。可以指定gropu任务超时的时间，无论指定的超时时间和group中任务完成哪个先到，dispatch\_group\_wait函数都会执行并有返回值。返回值为0即指定时间内任务全部完成，不为0则已超时，任务继续。
在dispatch\_group\_wait指定超时时间或group任务完成之前，执行dispatch\_group\_wait函数的当前线程阻塞。推荐使用dispatch\_group\_notify函数追加结束任务到队列中，因为 dispatch\_group\_notify函数可以简化源代码。


```

// 指定超时时间为2秒
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC));
long result = dispatch_group_wait(group, time);
if (result == 0) {
	NSLog(@"未超时，任务已经完成");
} else {
   NSLog(@"已超时，任务仍在继续");
}
    
```

**4.Dispatch\_barrier\_async**

避免数据竞争的思路：在写入处理结束之前，读取处理不可执行，写入处理追加到串行队列中，为了提高效率，读取处理追加到并发队列中。

GCD 提供更高效的方法：dispatch\_barrier\_async函数，该函数如同栅栏一般，使用并发队列和dispatch_barrier_async函数可实现高效率的数据访问和文件访问。

```
- (void)dispatch_barrier {
    void (^blk_reading) (void) = ^{
        for (NSInteger i = 0; i < 10000; i ++) {

        }
        NSLog(@"读取操作");
    };

    void (^blk_writing) (void) = ^{
        for (NSInteger i = 0; i < 10000; i ++) {

        }
        NSLog(@"写入操作");
    };

    dispatch_queue_t queue = dispatch_queue_create("com.example.concurrentQueue", DISPATCH_QUEUE_CONCURRENT);

    dispatch_async(queue, blk_reading);
    dispatch_async(queue, blk_reading);
    dispatch_async(queue, blk_reading);
    dispatch_barrier_async(queue, blk_writing);
    dispatch_async(queue, blk_reading);
    dispatch_async(queue, blk_reading);
    dispatch_async(queue, blk_reading);
}
```
输出：

```
读取操作
读取操作
读取操作
写入操作
读取操作
读取操作
读取操作

```

**5.Dispatch_sync**

与dispatch\_group\_wait相似，dispatch_sync的“等待”意味着阻塞当前线程，直到任务执行完毕，也可以说是简易版的dispatch\_group\_wait。

适用于在主线程中使用其他线程执行任务，任务结束后使用所得到的结果。

```
- (void)dispatch_sync_0 {
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_sync(queue, ^{
        NSLog(@"同步处理");
    });
}
```

注意：由于dispatch_sync会阻塞当前线程，使用不当会引起死锁，以下两例都会死锁。

```
- (void)dispatch_sync_1 {

    dispatch_queue_t queue = dispatch_get_main_queue();

    dispatch_sync(queue, ^{

        NSLog(@"同步处理");
    });
}

- (void)dispatch_sync_2 {

    // 每个串行队列都会对应一个线程
    dispatch_queue_t queue = dispatch_queue_create("com.example.serialQueue", DISPATCH_QUEUE_SERIAL);

    dispatch_async(queue, ^{
        dispatch_sync(queue, ^{
            NSLog(@"同步处理");
        });
    });
}
```

**6.Dispatch_apply**

dispatch_apply函数按照指定的次数将Block任务追加到指定的队列中，等待任务完成再执行其他操作。与 dispatch_sync一样，dispatch_apply也会阻塞线程。

```
- (void)dispatch_apply_1 {
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

    /**
     * param1 次数
     * param2 指定的队列
     * param3 带参数的Block
     */
    dispatch_apply(10, queue, ^(size_t index) {
        NSLog(@"任务%zu完成", index);
    });
    NSLog(@"全部完成");
}

- (void)dispatch_apply_2 {
    NSArray *array = @[@"任务1", @"任务2", @"任务3", @"任务4", @"任务5", @"任务6", @"任务7", @"任务8"];

    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_async(queue, ^{
        dispatch_apply([array count], queue, ^(size_t index) {
            // 处理任务
            NSLog(@"%@", [array objectAtIndex:index]);
        });
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"任务完成，更新UI");
        });
    });
}
```

**7.Dispatch_suspend / dispatch_resume**

这些操作不影响已经执行的任务。挂起后，队列中未执行的任务会停止，恢复后这些任务会继续执行。

**8.Dispatch Semaphore**

信号量用于对资源进行加锁操作。

```
- (void)dispatch_semaphore_1 {

    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);

    NSMutableArray *array = [[NSMutableArray alloc] init];

    for (NSInteger i = 0; i < 10000; i ++) {

        dispatch_async(queue, ^{

            dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);

            // 模拟数据写入操作
            [array addObject:@(i)];

            dispatch_semaphore_signal(semaphore);
        });
    }
}
```

信号量用于链式请求，限制一个请求完成后再去执行下一个。

```
- (void)dispatch_semaphore_2 {

    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{

        NSArray *array = @[@"1", @"2", @"3", @"4", @"5"];

        // 初始化信号量为0
        dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);

        [array enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {

            [self requestWithCompletion:^(NSDictionary *dict) {

                NSLog(@"%@-%@", dict[@"message"], obj);

                dispatch_semaphore_signal(semaphore);
            }];

            dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        }];
    });
    
- (void)requestWithCompletion:(void(^)(NSDictionary *dict))completion {
	dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        //模拟网络请求
        sleep(2);
        !completion ? nil : completion(@{@"message":@"任务完成"});
    });
}
```

**9.Dispatch_once**

dispatch_once函数能保证应用程序中任务只执行一次，该代码在多线程环境下执行可保证百分之百安全。常用于生成单例。