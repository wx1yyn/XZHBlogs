---
layout: post
title: "Runtime Part7 object_getClass()与[object class]的区别"
date: 2016-05-17 17:53:35 +0800
comments: true
categories: 
---

###刚好在看KVO的东西的时候，忽然遇到一个问题: `object_getClass(object)与[object class]返回的Class有什么区别？`一时间真的说不出来有什么区别，去补补...

***

###之前的runtime文章记录过: `类 Class` 、`元类 Meta-Class` 、`对象 object` 三者之间的关系

![](http://i3.piimg.com/f0be13b08322042d.png)

![](http://i2.piimg.com/02e5f1ad0ed1a2e4.png)

- 对象的 isa指针 指向 所属类

- 每一个类，都有一个 `isa指针` 指向一个 `唯一的 MetaClass`

- 每一个Meta Class，拥有的 `isa指针` 都直接指向 `顶层 NSObject Meta Class`

- 最上层的Meta Class（Meta NSObject）的`isa指针`指向自己，形成一个 `回路`

- 每一个`类本身` 的 `super_class指针` 指向其父类，如果该类为根类则值为 NULL

- 每一个 `MetaClass` 的 `super_class指针` 都指向其 `Super MetaClass`
	- 特别情况 >>> 最顶层 NSObject Meta Class 的 `super_class指针`，指向的却是 `NSObject 本身 Class`

***

###现在应该可以清晰的明白: Object（实例），Class（类）,Metaclass（元类），Rootclass(根类)，Rootclass‘s metaclass(根元类) 这些东西

- 对于NSObject的所有`子类`
	- Object（实例）
	- Class（类本身）
	- Metaclass（元类）

- 对于NSObject类自己 
	- Rootclass(根类本身)
	- Rootclass‘s metaclass(根元类)

- 类的一个对象: 类本身 的一个对象（可以单例、可以多例）

- `类本身`、`元类` 	
	- 根据注册一个类使用的api函数: `objc_registerClassPair()`来看表面意思是说注册`一对class`
		- 一个类 就是 类本身（我们经常调用的类）
		- 一个类 确实 元类（由苹果自己创建）
	- 类本身: 是objc_class结构体的一个单例
	- 元类: 也是objc_class结构体的一个单例

OK，基础东西搞明白再开始看有什么区别？

***

###当同时对一个`NSObject对象`作用时

```objc
Person *person1 = [Person new];
Person *person2 = [Person new];
Person *person3 = person1;
    
//object_getClass()获取Class
NSLog(@"object_getClass >>> person1: %p", object_getClass(person1));
NSLog(@"object_getClass >>> person2: %p", object_getClass(person2));
NSLog(@"object_getClass >>> person3: %p", object_getClass(person3));
    
//[object class]获取Class
NSLog(@"[object class] >>> person1: %p", [person1 class]);
NSLog(@"[object class] >>> person2: %p", [person2 class]);
NSLog(@"[object class] >>> person3: %p", [person3 class]);
```

输出如下:

```
2016-05-17 18:17:24.793 demo[18638:235175] object_getClass >>> person1: 0x10b966d00
2016-05-17 18:17:24.793 demo[18638:235175] object_getClass >>> person2: 0x10b966d00
2016-05-17 18:17:24.793 demo[18638:235175] object_getClass >>> person3: 0x10b966d00
2016-05-17 18:17:24.793 demo[18638:235175] [object class] >>> person1: 0x10b966d00
2016-05-17 18:17:24.794 demo[18638:235175] [object class] >>> person2: 0x10b966d00
2016-05-17 18:17:24.794 demo[18638:235175] [object class] >>> person3: 0x10b966d00
```

从输出可以得到两点:

- 不同的NSObject对象，得到的是`同一个objc_class结构体的实例`.
- 当对一个`NSObject对象`作用是，object_getClass()与[object class]返回的`objc_class结构体实例`都是用一个地址.


****

###当同时对一个`类`作用时

注意，object_getClass()方法接收的参数，如下两种形式调用，第一种是编译器报错的.


```objc
//object_getClass()获取Class
NSLog(@"object_getClass >>> person1: %p", object_getClass(Person));//这里会报错
    
//[object class]获取Class
NSLog(@"[object class] >>> person1: %p", [Person class]);// 这句是正确的，+[NSObject class] 与 -[NSObejct class]
```

看下object_getClass()方法的声明:

```objc
/** 
 * Returns the class of an object.
 * 
 * @param obj The object you want to inspect.
 * 
 * @return The class object of which \e object is an instance, 
 *  or \c Nil if \e object is \c nil.
 */
OBJC_EXPORT Class object_getClass(id obj) 
```

对于这个方法参数我有点小迷糊，`一个NSObject类`为啥不能当做object_getClass()的参数了？

```objc
// 错误
object_getClass(Person);

// 正确
object_getClass([Person class]);
```

如上代码第一种直接传入一个`NSObject类`作为参数就会编译器报错，但是第二种`获取类的objc_class结构体实例`之后再传进去就通过编译了。但是NSObject类不也是对象吗？为啥就不能当做参数传入了？

####想了几分钟，突然灵感来临想到了几点原因:

- Runtime这一套类库，都是基于`c语言代码`实现的

- iOS Objective-C程序最终也是编译为`c语言的代码`，再编译生成Mach-O格式的可执行文件

- 而`+[NSObject类 class]`获取的是objc_class结构体的单例对象
	- 而获取到的这个`objc_class 结构体单例`，就是这个NSObejct类在`运行阶段`时系统内存中的一个内存块
	- 也就是说这个`objc_class 结构体单例`，才是NSObject类在内存中的最终体现形式

证明NSObject类对应的在运行时阶段的objc_class结构体实例是单例

```objc
NSLog(@"%p", [Person class]);
NSLog(@"%p", [[Person new] class]);
NSLog(@"%p", [[Person new] class]);
NSLog(@"%p", [[Person new] class]);
```

输出如下

```
2016-05-17 22:53:07.339 demo[24532:314044] 0x10ee07de0
2016-05-17 22:53:07.340 demo[24532:314044] 0x10ee07de0
2016-05-17 22:53:07.340 demo[24532:314044] 0x10ee07de0
2016-05-17 22:53:07.340 demo[24532:314044] 0x10ee07de0
```

从输出的地址上看，都是指向同一个地址，所以NSObject类对应的objc_class结构体实例确实是单例，全局只存在一个内存空间来存放.
	
####结合上面几点，我想可以确切的解释为:

- `NSObject类` 并不是一个 `真正的对象`，只是我们可以看做成一个对象:

```
- 1、`代码编写阶段`: NSObject类，还不是一个对象（没有分配内存空间），而是一个死的
- 2、 `OC程序编译成c程序阶段`: 编译器会将我们所写的NSObejct类（基于OC的类），全部编译为 `objc_class 结构体`（基于c语言表达的类结构）
- 3、`程序运行阶段`: 就会给这些`objc_class 结构体`分配对应的唯一内存空间来存放
- 4、所以在运行阶段，实例化出来的一个具体的 objc_class 结构体实例，才能代表一个我们所编写的NSObject类
```

- 实际上所说的 `类对象` >>> NSObject类在运行时内存中，所对应的一个`objc_class结构体`实例，而不应该是NSObject这个OC类.


#### 一个 objc_class结构体 实例 >>> 在运行阶段表示一个我们所编写的NSObject类

```objc
struct objc_class {			
	struct objc_class *isa;	
	struct objc_class *super_class;	
	const char *name;		
	long version;
	long info;
	long instance_size;
	struct objc_ivar_list *ivars;

	struct objc_method_list **methodLists;

	struct objc_cache *cache;
 	struct objc_protocol_list *protocols;
};
```

http://opensource.apple.com/source/objc4/objc4-237/runtime/objc-class.h


明白这些之后，继续看当同时对一个`类`作用时，`object_getClass()`与`[NSObject class]`的区别:

```objc
//object_getClass()获取Class
NSLog(@"object_getClass >>> person1: %p", object_getClass([Person class]));

//[object class]获取Class
NSLog(@"[object class] >>> person2: %p", [Person class]);
```

输出如下 

```
2016-05-17 22:38:10.457 demo[23737:301660] object_getClass >>> person1: 0x10adadd10
2016-05-17 22:38:10.458 demo[23737:301660] [object class] >>> person2: 0x10adadd38
```

从地址输出看，在对`类对象`进行操作时，objc_getClass() 与 +[NSObject class]得到的`objc_class`结构体实例是`不同`的，为什么了？

我想是时候看看 objc_getClass()和 +[NSObject class]的源码实现了...

#### object_getClass()方法的源码实现

```objc
Class object_getClass(id obj)
{
    if (obj) return obj->getIsa();//返回isa指针所指向的objc_class结构体实例
    else return Nil;
}
```

http://opensource.apple.com/source/objc4/objc4-680/runtime/objc-class.mm

#### +[NSObject class] 与 -[NSObject class]的源码实现

```objc
+ (Class)class {
    return self;//返回类本身的objc_class实例
}

- (Class)class {
    return object_getClass(self);//返回对象->isa对于的objc_class实例，实际上就是类本身的objc_class实例
}
```

http://opensource.apple.com/source/objc4/objc4-680/runtime/NSObject.mm

####再结合前面 NSObject对象、NSObject类、Meta类三者关系图

![](http://i2.piimg.com/02e5f1ad0ed1a2e4.png)

从上图可以看出:

- `对象`的isa指针 >>>> 指向NSObject类对应的`objc_class结构体`实例

- `NSObject类`的isa指针 >>>> 却是指向NSObject类的`另一半Meta元类`所对应的`objc_class结构体`实例

####再回头看`object_getClass([Person class])` 与 `[Person class]` 、 `[Person对象 class]`的区别:

- object_getClass([Person class])
	
	- 首先传入的是Person这个NSObject类所对应的objc_class结构体实例
	- objc_getClass()方法获取到传入的`objc_class实例->isa`所指向的`另一个objc_class结构体实例`
	- 而被指向的另一个objc_class结构体实例 >>> Peson类的`元类`
	- **所以最终得到的是Person元类的objc_class结构体实例**

- [Person class]
	- **返回的就是Person这个NSObject类对应的objc_class结构体实例**

- [Person对象 class]
	- 首先根据对象->isa指针找到所属类Person对应的objc_class结构体实例
	- 然后再返回找到的objc_class结构体实例

####所以当对于一个`NSObject类`时，有如下不等式永远都是成立的

```
object_getClass([NSObject类 class]) != [NSObject类 class] 
object_getClass([NSObject类 class]) != [NSObject对象 class]
```

因为

```
object_getClass([NSObject类 class]) >>> Meta Class
```

```
[NSObject类 class] == [NSObject对象 class] >>> Class本身
```


****

###最后再小结下他们的实现

#### objc_getClass()方法的源码实现

```objc
Class object_getClass(id obj)
{
    if (obj) return obj->getIsa();
    else return Nil;
}
```

**读取传入对象的isa指针指向的另一个objc_class结构体实例**


#### +[NSObject class] 与 -[NSObject class]的源码实现

```objc
+ (Class)class {
    return self;
}
```

**类方法、直接返回的就是当前NSObject类对应的objc_class结构体实例**

```objc
- (Class)class {
    return object_getClass(self);
}
```

**实例方法、返回当前对象->isa指向的objc_class结构体实例**


###那么我想可以回答本篇文章开头的疑问了: object_getClass()与[object class] 有什么区别？

- `object_getClass()`方法，获取类对象的isa指针指向的另外一个objc_class结构体实例

- 而对于`NSObject类`的isa指针，指向的是`Meta元类`，已经不是`NSObject类本身`了

- [Person class]与[Person对象 class]获取都是`Person类本身`的objc_class结构体实例

- 所以当两种方法同时去操作一个`类`时，得到的objc_class结构体实例是`不一样`的
	- 前者是 `Meta类` 的objc_class结构体实例
	- 后者是 `类本身` 的objc_class结构体实例
	- 所以地址是不相等的

***

###还有一个与`object_getClass()`类似的方法: `objc_getClass()`

```c
// 接收一个c字符串
Class objc_getClass(const char *name)
```

以一个小例子看其区别:

```objc
NSLog(@"%p", [Person class]);
NSLog(@"%p", object_getClass([Person class]));
NSLog(@"%p", objc_getClass("Person"));
```

结果输出

```
2016-06-22 16:12:29.007 demo[10437:1581222] 0x104e53c28
2016-06-22 16:12:29.008 demo[10437:1581222] 0x104e53c00
2016-06-22 16:12:29.008 demo[10437:1581222] 0x104e53c28
```

应该知道了吧..

***

###respondsToSelector 既可以判断类是否实现方法也可以判断对象是否实现方法

```objc
@implementation ViewController

+ (void)classMethod {
    
}

- (void)instanceMethod {
    
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

	BOOL flag1 = [self respondsToSelector:@selector(instanceMethod)];//YES
    BOOL flag2 = [[self class] respondsToSelector:@selector(instanceMethod)];//NO
    BOOL flag3 = [self respondsToSelector:@selector(classMethod)];//NO
    BOOL flag4 = [[self class] respondsToSelector:@selector(classMethod)];//YES
}

@end
```

输出

```
(BOOL) flag1 = YES
(BOOL) flag2 = NO
(BOOL) flag3 = NO
(BOOL) flag4 = YES
```

当`[self respondsToSelector:]`时，是向`self->isa`指向的`objc_class`实例中查询IMP。

而当`[[self class] respondsToSelector:]`时，是向`[self class]->isa`指向的`MetaClass`对应的`objc_class`实例中查询IMP。

所以既可以判断类是否实现方法也可以判断对象是否实现方法。

***

###学习资源

```
http://www.jianshu.com/p/ae5c32708bc6
```