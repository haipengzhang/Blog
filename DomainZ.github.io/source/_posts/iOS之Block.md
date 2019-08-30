---
title: iOS之Block
date: 2018-02-05 16:05:29
tags: iOS
---

Block用起来很方便，编译过程到底是怎么回事呢？

### Block的数据结构

```
struct Block_descriptor_1 {
    uintptr_t reserved;
    uintptr_t size;
};
 
struct Block_layout {
    void *isa;
    volatile int32_t flags; // contains ref count
    int32_t reserved; 
    void (*invoke)(void *, ...);
    struct Block_descriptor_1 *descriptor;
    // imported variables
};
```
Block编译之后成为一个结构体（实际上就是对象类型因为第一位是*isa），一个函数指针，指向的是编译过程中根据block中代码生成的Imp，最后面的是block中代码持有的变量(相当于对象的成员变量)；

### Block之变量

* block里面对内外部变量会将其对象化成成员变量不添加*__block*修饰只能是只读，用*__weak*修饰外部截取的变量retain count不会+1否则+1强持有；

* 能用*__block*来指定block中想变更值的截获的变量，在block从栈拷贝到堆的时候，截获的变量都会拷贝到堆上，*__block*修饰的变量为了能够实现栈和堆都可以正常读写到，栈区的该变量的*__forwardingk*指针会指向到已经拷贝到堆的变量，读写改变了通过*__forwarding*指针取值；

* 除了*__weak*也能用*__block*解决循环引用，只需要在block语句最会赋值为空；

### 栈上的Block栈堆转移
* 将栈上的Block复制到堆上，这样即使Block记述的变量作用域结束，堆上的Block还可以继续存在。

```
- (void)blockTest {
    id obj = [self getBlockArray];
    typedef void (^blk_t)(void);
    blk_t blk = (blk_t)[obj objectAtIndex:0];
    blk();
}

- (id)getBlockArray {
    int val = 10;
    return [[NSArray alloc] initWithObjects:^{NSLog(@"blk0:%d",val);},^{NSLog(@"blk1:%d",val);},nil];
    }
```
执行以上blockTest方法会出错，原因在于变量作用域。
 [^{NSLog(@"blk0:%d",val);} copy]移到堆解决问题；

### 栈上的Block复制到堆上的时机
* 调用block的copy方法时；
* block作为函数返回值返回时；
* 将block赋值给*__strong*修饰的id类型或block类型变量时；
* 在方法名中含有usingBlock的Cocoa框架方法或GCD的API中传递Block时；



