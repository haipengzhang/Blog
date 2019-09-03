---
title: iOS之Runtime相关
date: 2019-09-03 10:23:36
tags: runtime
categories: iOS

---

### Runtime常见概念
#### 类
##### 类的结构
类对象(Class)是由程序员定义并在运行时由编译器创建的，它没有自己的实例变量，这里需要注意的是类的成员变量和实例方法列表是属于实例对象的，但其存储于类对象当中的。

	typedef struct objc_class *Class;
	
	struct objc_class {
    	Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
	
		#if !__OBJC2__
    	
    	Class _Nullable super_class OBJC2_UNAVAILABLE;
    	const char * _Nonnull name OBJC2_UNAVAILABLE;
    	long version OBJC2_UNAVAILABLE;
    	long info OBJC2_UNAVAILABLE;
    	long instance_size OBJC2_UNAVAILABLE;
    	struct objc_ivar_list * _Nullable ivars OBJC2_UNAVAILABLE;
    	struct objc_method_list * _Nullable * _Nullable methodLists OBJC2_UNAVAILABLE;
    	struct objc_cache * _Nonnull cache OBJC2_UNAVAILABLE;
    	struct objc_protocol_list * _Nullable protocols OBJC2_UNAVAILABLE;
	
		#endif

	} OBJC2_UNAVAILABLE;

{% asset_img isa.png isa %}

- isa：简而言之就是**Objc对象的isa指针指向的是类对象**。类对象包含isa和superclass指针，它们的isa指向的是metaclass，它们的superclass指向父类。metaclass的isa指向它的基类；
- cache：用于缓存最近使用的方法。一个接收者对象接收到一个消息时，它会根据isa指针去查找能够响应这个消息的对象。在实际使用中，这个对象只有一部分方法是常用的，很多方法其实很少用或者根本用不上。这种情况下，如果每次消息来时，我们都是methodLists中遍历一遍，性能势必很差。这时，cache就派上用场了。在我们每次调用过一个方法后，这个方法就会被缓存到cache列表中，下次调用的时候runtime就会优先去cache中查找，如果cache没有，才去methodLists中查找方法。这样，对于那些经常用到的方法的调用，但提高了调用的效率。

##### 获取类名
	const char * class_getName ( Class cls ); 
##### 动态创建类
	// 创建一个新类和元类
	Class objc_allocateClassPair ( Class superclass, const char *name, size_t extraBytes ); //如果创建的是root class，则superclass为Nil。extraBytes通常为0

	// 销毁一个类及其相关联的类
	void objc_disposeClassPair ( Class cls ); //在运行中还存在或存在子类实例，就不能够调用这个。

	// 在应用中注册由objc_allocateClassPair创建的类
	void objc_registerClassPair ( Class cls ); //创建了新类后，然后使用class_addMethod，class_addIvar函数为新类添加方法，实例变量和属性后再调用这个来注册类，再之后就能够用了。
	
#### 对象
实例对象是我们对类对象alloc或者new操作时所创建的，在这个过程中会拷贝实例所属的类的成员变量，但并不拷贝类定义的方法。对象最重要的是可以给其发送消息，调用实例方法时，系统会根据实例的isa指针去类的方法列表及父类的方法列表中寻找与消息对应的selector指向的方法。

	/// Represents an instance of a class.
	struct objc_object {
    	Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
	};
##### 对对象的类操作
	// 返回给定对象的类名
	const char * object_getClassName ( id obj );
	// 返回对象的类
	Class object_getClass ( id obj );
	// 设置对象的类
	Class object_setClass ( id obj, Class cls );
	
##### 获取对象的类定义
	// 获取已注册的类定义的列表
	int objc_getClassList ( Class *buffer, int bufferCount );

	// 创建并返回一个指向所有已注册类的指针列表
	Class * objc_copyClassList ( unsigned int *outCount );

	// 返回指定类的类定义
	Class objc_lookUpClass ( const char *name );
	Class objc_getClass ( const char *name );
	Class objc_getRequiredClass ( const char *name );
	
	// 返回指定类的元类
	Class objc_getMetaClass ( const char *name );
	
##### 动态创建对象
	// 创建类实例
	id class_createInstance ( Class cls, size_t extraBytes ); //会在heap里给类分配内存。这个方法和+alloc方法类似。

	// 在指定位置创建类实例
	id objc_constructInstance ( Class cls, void *bytes ); 

	// 销毁类实例
	void * objc_destructInstance ( id obj ); //不会释放移除任何相关引用
	
#### Metaclass
元类(Metaclass)就是类对象的类，每个类都有自己的元类，也就是objc_class结构体里面isa指针所指向的类. Objective-C的类方法是使用元类的根本原因，因为其中存储着对应的类对象调用的方法即**类方法**。

#### 属性
在Objective-C中，属性(property)和成员变量是不同的。那么，属性的本质是什么？它和成员变量之间有什么区别？简单来说属性是添加了存取方法的成员变量，也就是:

	@property = ivar + getter + setter;
	


### 消息转发
#### objc_msgSend
	[array insertObject:foo atIndex:5];
	// 等价
	objc_msgSend(array, @selector(insertObject:atIndex:), foo, 5);	
0. 首先检查这个selector是不是要忽略;
1. 检测这个selector的target是不是nil，OC允许我们对一个nil对象执行任何方法不会Crash，因为运行时会被忽略掉;
2. 如果上面两步都通过了，就开始查找这个类的实现IMP，先从cache里查找，如果找到了就运行对应的函数去执行相应的代码;
3. 如果cache中没有找到就找类的方法列表中是否有对应的方法;
4. 如果类的方法列表中找不到就到父类的方法列表中查找，一直找到NSObject类为止;
5. 如果还是没找到就要开始进入动态方法解析和消息转发;

#### 动态方法解析：
{% asset_img msg_forwarding.png msg_forwarding %}
没有方法的实现，程序会在运行时挂掉并抛出 unrecognized selector sent to … 的异常。但在异常抛出前，Objective-C 的运行时会给你三次拯救程序的机会：

- Method resolution
- Fast forwarding
- Normal forwarding

0. Runtime 发送 +resolveInstanceMethod: 或者 +resolveClassMethod: 尝试去 resolve 这个消息；
1. 如果 resolve 方法返回 NO，Runtime 就发送 -forwardingTargetForSelector: 允许你把这个消息转发给另一个对象；
2. 如果没有新的目标对象返回， Runtime 就会发送-methodSignatureForSelector: 和 -forwardInvocation: 消息。你可以发送 -invokeWithTarget: 消息来手动转发消息或者发送 -doesNotRecognizeSelector: 抛出异常。

应用：防止程序崩溃机制；

#### 应用
Associate用来给instance加变量的方式，Method swizzling实现hook（插入编程），自实现block型kvo和notificationcenter。


