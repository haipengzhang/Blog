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
Block编译之后成为一个结构体（实际上就是对象类型因为第一位是*isa），一个函数指针，指向的是编译过程中根据block中代码生成的Imp，最后面的是block中代码持有的变量；

### Block Tips
* 正常一个对象的创建是在堆上开辟内存空间，block创建是在栈区，赋值过程中会被拷贝到堆区；

* __block定义的变量实际上是将其对象化，然后拷贝到堆区，然后在block里面强引用，从而可以实现对外部变量的操作；

```
//这个实际上是一个对象了，之后如果对i的操作都是对其堆区的数据操作
__block int i = -1;
```

* Block里面对象强引用，里面记得用weak；