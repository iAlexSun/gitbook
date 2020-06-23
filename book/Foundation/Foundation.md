## Foundation

- 1.`nil`、`NIL`、`NSNULL` 有什么区别？

  

- 2.如何实现一个线程安全的 `NSMutableArray`? 

  

- 3.如何定义一台 `iOS` 设备的唯一性? 

  IDFA, UDID

- 4.`atomic` 修饰的属性是绝对安全的吗？为什么？

  不是绝对安全的，只会保证该属性的setter，getter方法是线程安全的。如果在多线程中没有进行setter，getter方法依旧可能会出现崩溃情况，比如多线程中操作数组，可能会出现数组越界的情况。所以可以加锁或者一般读写的情况下使用`dispatch_barrier_sync`函数也可以避免多线程的资源竞争问题。详细的可以参考多线程问题

  ```objective-c
  - (void)dispatch_asyncArr {
          self.arr = [NSMutableArray array];
          dispatch_queue_t concurrent_queue = dispatch_queue_create("com.domain.queuename", DISPATCH_QUEUE_CONCURRENT);
          for (int i=0; i<10000; i++) {
              dispatch_async(concurrent_queue, ^{
                  [self.arr addObject:@(i)];
                  NSLog(@"%@---%d---",[NSThread currentThread],i);
              });
          };
  }
  ```

  

- 5.实现 `isEqual` 和 `hash` 方法时要注意什么？

  

- 6.`id` 和 `instanceType` 有什么区别？

  

- 7.简述事件传递、事件响应机制。

  

- 8.说一下对 `Super` 关键字的理解

  `super`是一个关键字，一般情况都是`[super message]`去寻找父类的实现的方法，但是`[super class]`并不是父类的 `class` 类型不等于 `[self.superclass class]` ，这里的原因是因为 `super`会去寻找系统实现的`objc_msgsendSuper()`，objc_msgSendSuper源代码如下

  ```objective-c
  OBJC_EXPORT void objc_msgSendSuper(void /* struct objc_super *super, SEL op, ... */ )
  /// Specifies the superclass of an instance. 
  struct objc_super {
      /// Specifies an instance of a class.
      __unsafe_unretained id receiver;
  
      /// Specifies the particular superclass of the instance to message. 
  #if !defined(__cplusplus)  &&  !__OBJC2__
      /* For compatibility with old objc-runtime.h header */
      __unsafe_unretained Class class;
  #else
      __unsafe_unretained Class super_class;
  #endif
      /* super_class is the first class to search */
  };
  ```

  在objc_msgSendSuper方法中，第一个参数是一个objc_super的结构体，这个结构体里面有两个变量，一个是接收消息的receiver，一个是当前类的父类super_class。

  入院考试第一题错误的原因就在这里，误认为[super class]是调用的[super_class class]。

  objc_msgSendSuper的工作原理应该是这样的:
   从objc_super结构体指向的superClass父类的方法列表开始查找selector，找到后以objc->receiver去调用父类的这个selector。注意，最后的调用者是objc->receiver，而不是super_class！

  那么objc_msgSendSuper最后就转变成

  ```objective-c
  // 注意这里是从父类开始msgSend，而不是从本类开始，谢谢@Josscii 和他同事共同指点出此处描述的不妥。
  objc_msgSend(objc_super->receiver, @selector(class))
  
  /// Specifies an instance of a class.  这是类的一个实例
      __unsafe_unretained id receiver;   
  // 由于是实例调用，所以是减号方法
  - (Class)class {
      return object_getClass(self);
  }
  ```

- 9.了解 `逆变` 和 `协变` 吗？

  

- 10.`@synthesize` 和 `@dynamic` 分别有什么作用？

  `@synthesize`自动生成setter getter方法，默认情况下`synthesize` xxx =_xxx

  `@dynamic`需要手动生成setter getter方法

  

- 11.`Obj-C` 中的反射机制了解吗？

  

- 12.`typeof` 和 `__typeof`，`__typeof__` 的区别?

  

- 13.如何判断一个文件在沙盒中是否存在？

  

- 14.头文件导入的方式？ 

  

- 15.如何将 `Obj-C` 代码改变为 `C++/C` 的代码？

  

- 16.知不知道在哪里下载苹果的源代码？

  

- 17.`objc_getClass()`、`object_getClass()`、`Class` 这三个方法用来获取类对象有什么不同？