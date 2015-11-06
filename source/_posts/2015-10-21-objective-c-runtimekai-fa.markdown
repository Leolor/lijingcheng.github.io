---
layout: post
title: "Objective-C Runtime开发"
date: 2015-9-21 14:30:27 +0800
comments: true
categories: 
---
<br/>

{% img left http://7x00ed.com1.z0.glb.clouddn.com/runtime.jpg 150 150 "" "" %}
>**Runtime - *使用C和汇编实现的运行时代码库，Objective-C中有很多语言特性都是通过它来实现。***   
了解Runtime开发可以帮助我们更灵活的使用Objective-C这门语言，我们可以将程序功能推迟到运行时再去决定怎么做，还可以利用Runtime来解决项目开发中的一些设计和技术问题，使开发过程更加具有灵活性。

<br/>
<!-- more -->

#消息传递(Messaging)
Objective-C对于调用对象的某个方法这种行为叫做给对象发送消息，对象在运行时要执行什么方法不是在编译时决定，而是在程序运行时才决定，runtime会根据消息接收者是否能响应消息而做出不同的反应，消息也许会由该指定对象处理，也可以由其它对象处理，还可以让对象执行别的方法或者不处理。下面我们来了解一下这个过程：

- 我们写一个给对象发送消息的代码  
``` objc
[array insertObject:obj atIndex:5];
```
- 编译器首先会将上面代码翻译成这种样子  
``` objc
objc_msgSend(array, @selector(insertObject:atIndex:), obj, 5);
```  

- 系统在运行时会通过array对象的isa指针找到对应的class。
- 在class的cache方法列表中用SEL去找对应method，如果找不到便去class的方法列表中去找
- 如果在方法列表中也找不对对应method时，便沿着继承体系继续向上查找
	- 找到后将method放入cache，以便下次能快速定位，然后再去执行method的IMP
	- 找不到时系统便报错：unrecognized selector sent to insertObject:atIndex:
	
<br/>
**Runtime提供了三种方法避免因为找不到方法而崩溃** 

- 当找不到方法实现时，Runtime会先发送"+resolveInstanceMethod:"或"+resolveClassMethod:"消息，我们可以重写它然后为对象指定一个处理方法。

``` objc
void dynamicXXXMethod(id obj, SEL _cmd)  
{
    NSLog(@"ok...");
}

+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
    if(aSEL == @selector(xxx:)) {
        class_addMethod([self class], aSEL, (IMP)dynamicXXXMethod, "v@:");
        
        return YES;
    }
    
    return [super resolveInstanceMethod];
}

```
> class_addMethod方法的最后一个参数用来指定所添加方法的参数及返回值，叫[Type Encodings](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)。

- 如果resolve方法返回"NO"，Runtime会发送"-forwardingTargetForSelector:"消息，允许我们将消息转发给能处理它的其它对象。
``` objc
- (id)forwardingTargetForSelector:(SEL)aSelector
{
    if(aSelector == @selector(xxx:)){
        return otherObject;
    }
    return [super forwardingTargetForSelector:aSelector];
}

```

- 当"-forwardingTargetForSelector:"返回"nil"时，Runtime会发送"-methodSignatureForSelector:"和"-forwardInvocation:"消息。我们可以选择忽略消息、抛出异常、将消息转由当前对象或其它对象的任意消息来处理。
``` objc
//根据SEL生成NSInvocation对象，然后再由"-forwardInvocation:"方法进行转发。
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
{
    NSMethodSignature *signature = [super methodSignatureForSelector:aSelector];

    if (!signature) {
        signature = [otherObject instanceMethodSignatureForSelector:aSelector];
    }

    return signature;
}

- (void)forwardInvocation:(NSInvocation *)invocation
{
    SEL sel = invocation.selector;

    if([otherObject respondsToSelector:sel]) {
        [invocation invokeWithTarget:otherObject];//转发消息
    } 
    else {
        [self doesNotRecognizeSelector:sel];//抛出异常
    }
}
```
<br/>

#KVO
当我们为对象添加观察者后，Runtime会在运行时创建这个对象所在类的子类，然后重写监听属性的set方法并在方法中调用"-willChangeValueForKey:"和"-didChangeValueForKey:"来通知观察者，当移除观察者后，Runtime便会将这个子类删除。

<br/>

#关联对象(Associated Objects)
在Category中可以为类添加实例方法或类方法，但是不支持添加实例变量，所以即使我们在category中为类添加了property，也不能直接使用它，Runtime可以解决这个问题，我们只需要定义一个指针，然后通过"objc_setAssociatedObject"方法将指针与对象进行关联并指定内存管理方式。具体代码如下：  

``` objc
@interface NSObject (JC)

@property (nonatomic, copy) NSString *ID;

@end


@implementation NSObject (JC)

static const void *IDKey;

- (NSString *)ID
{
    return objc_getAssociatedObject(self, &IDKey);
}

- (void)setID:(NSString *)ID
{
    objc_setAssociatedObject(self, &IDKey, ID, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

@end  
```  
<br/>

#AOP(Method Swizzling)
我们可以通过继承、Category、AOP方式来扩展类的功能。

- 继承比较适合在设计底层代码架构时使用，不适当的使用会让代码看起来很啰嗦，并且增加维护难度。
- Category适合为现有类添加方法。
- 当需要修改现有类的方法并且拿不到源码时，继承和AOP都能解决问题，但是用AOP来解决代码耦合度更低。其实就算能拿到源码，往往直接去改源码也不是个好办法。

<br/>
在Objective-C中，可以通过"Method Swizzling"技术来实现AOP，下面我们通过交换两个方法的实现代码来向已存在的方法中添加其它功能。

``` objc  
#import <objc/runtime.h> 
 
@implementation UIViewController (Tracking) 
 
+ (void)load 
{ 
    static dispatch_once_t onceToken; 
    dispatch_once(&onceToken, ^{ 
        Class aClass = [self class]; 
 
        SEL originalSelector = @selector(viewWillAppear:); 
        SEL swizzledSelector = @selector(swizzled_viewWillAppear:); 
 
        Method originalMethod = class_getInstanceMethod(aClass, originalSelector); 
        Method swizzledMethod = class_getInstanceMethod(aClass, swizzledSelector); 
        
        // 如果要对类方法进行交换，使用下面注释的代码
        // Class aClass = object_getClass((id)self);
        // 
        // Method originalMethod = class_getClassMethod(aClass, originalSelector);
        // Method swizzledMethod = class_getClassMethod(aClass, swizzledSelector);
 
 		//交换两个方法的实现
        BOOL didAddMethod = class_addMethod(aClass, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod)); 
 
        if (didAddMethod) { 
            class_replaceMethod(aClass, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod)); 
        } else { 
            method_exchangeImplementations(originalMethod, swizzledMethod); 
        } 
    }); 
} 
 
#pragma mark - Method Swizzling 

//由于方法实现已经被交换，所以系统在调用"viewWillAppear:"时，实际上会调用"swizzled_viewWillAppear:"
- (void)swizzled_viewWillAppear:(BOOL)animated
{ 
	//下面代码表面上看起来会引起递归调用，由于函数实现已经被交换，实际上会调用"viewWillAppear:"
   [self swizzled_viewWillAppear:animated]; 

	//在原有基础上添加其它功能(写日志等)
} 
 
@end
```  
<br/>

#其它
我们可以通过Runtime特性来获得类的所有属性名称和类型，然后再通过KVC将Json中的值填充给该类的对象。还可以在程序运行时为类添加方法或替换方法从而使对象能够更灵活的根据需要来选择实现方法。总之Runtime库就象一堆积木，只要发挥想象力便能实现各种各样的功能，但前提是你需要了解它。

