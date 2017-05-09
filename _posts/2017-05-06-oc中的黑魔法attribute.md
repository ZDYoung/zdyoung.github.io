---
layout:     post
title:      oc中的黑魔法attribute
date:       2017-05-06
summary:    attribute是编译器为我们提供的一个指令，在编程过程中善用它能为我们带来意想不到的好处。
categories: objective-c attribute 
---

标签： objective-c attribute

--- 

## 前言

__attribute__其实是一个编译器指令，用于指定一些声明中特性包括函数声明和变量声明，这样一来可以编译器可以执行一些错误检查或是进行一些高级的优化。说的直白一点就是对声明加一点限制，拿函数而言，可以用__attribute__来限制形参不为空，这样一来，可以保证在编程的过程，函数调用形参一定是不为空的。这只是其中一个用法。这个指令对于编写框架以及更新框架用的多，在实际的编程中用的不多，下面列出的都是一些用法。

## 正文

对这个指令还比较陌生的程序员，那么在编程时候，肯定遇到过这样的情况：当调用一人API时，编译器发出警告说这个API在某某版本中已经被废弃，现在用某某API代替。这其实就是__attribute__的一种用法。

#### 用法

__attribute__后面是由两组圆括号，括号内的属性由逗号分割，该指令放在函数，变量和类型声明之后。

1.<font color="#FF6100" >__attribute__((cleanup(...)))</font>，用于修饰一个变量，在它的作用域结束时可以自动执行一个指定的方法，如：
```objc
// 指定一个cleanup方法，注意入参是所修饰变量的地址，类型要一样
// 对于指向objc对象的指针(id *)，如果不强制声明__strong默认是__autoreleasing，造成类型不匹配
static void stringCleanUp NSString *__strong *string) {
    NSLog(@"%@", *string);
}
// 在某个方法中：
{
     NSString *__strong string __attribute__((cleanup(stringCleanUp))) = @"hello";
} // 当运行到这个作用域结束时，自动调用stringCleanUp
```
所谓作用域结束，包括大括号结束、return、goto、break、exception等各种情况。
当然，可以修饰的变量不止NSString，自定义Class或基本类型都是可以的。

这个指令可以修饰变量，当然也就可以修饰block：
```objc
// void(^block)(void)的指针是void(^*block)(void)
static void blockCleanUp(__strong void(^*block)(void)) {
    (*block)();
}
```
然后我们在一个作用域里来定义一个block变量：
```objc
{
   // 加了个`unused`的attribute用来消除`unused variable`的warning
    __strong void(^block)(void) __attribute__((cleanup(blockCleanUp), unused)) = ^{
        NSLog(@"I'm dying...");
    };
} //执行完这一行后就会输出"I'm dying..."
```

2.<font color="#FF6100" >__attribute__((noreturn))</font>，它用于指定函数是不允许有返回值的。
```objc
- (void)noReturn __attribute__((noreturn))
{
    NSLog(@"hello");
}
```
这个笔者没用到过。

3.<font color="#FF6100" >__attribute__((deprecated))</font>,这个可能是大家见的最多了，苹果也经常干这个事，每隔一两个版本可能就会有那么几个API被废弃，然后提示我们用哪个API代替，在编译的时候就会发出警告。
```objc
- (void)deprecatedMethodWithMessage __attribute__((deprecated("this method was deprecated in MyApp.app version 5.0.2, use newMethod instead")))
{
    NSLog(@"his method was deprecated in MyApp.app version 5.0.2, use newMethod instead");
}//调用这个方法的时候会发出警告信息，deprecated后面的参数可要可不要。
```

4.<font color="#FF6100" >__attribute__((warn_unused_result))</font>
```objc
- (NSString *)printMsg:(NSString *)str __attribute__((warn_unused_result))
{
    return str;
}
//这样调用这个函数会报警告信息,提示用户返回值没有被使用，在一些返回值很重要的函数里，这个很有用
[self printMsg:@"msg"];
```

5.<font color="#FF6100" >__attribute__((unavailable))</font>，这个和deprecated有很大的不一样，如果对函数加上这个，那么这个函数是禁止使用的。
```objc
- (instancetype)init __attribute__ ((unavailable("这个函数不能用！")));
```
如果强行使用这个函数，编译是通不过的。这一种做法可禁止使用默认的初始化方法生成类的实例对象，只能使用用户提供的初始化方法。

6.<font color="#FF6100" >__attribute__((overloadable))</font>，这个可以用来实现函数的重载。这里先来说一下两个概念：重载和覆盖。
1.  重载：它作用域是在一个类里面，函数名相同，参数不同（参数类型不同，或者参数个数不同），virtual关键字可有可无。

2.  覆盖：作用域不同，一个是在基类中一个是在派生类中，参数相同（参数个数和类型均相同），基类函数必须有virtual关键字（派生类可有可无，因为基类函数被声明为虚函数，派生类同名函数一定也是虚函数）

知道这个两个概念的区别，那么来讲讲objective-c，在这门语言中，是没有重载这个机制的，但是可以覆盖。使用overloadable后可以实现重载，但是不能像c++那样重载实例方法，只能重载c/c++的函数：
```objc
//调用以下函数时和c/c++调用的语法一样
void add(int num) __attribute__((overloadable))
{
    NSLog(@"Add Int %i",num);
}
void add(NSString * num) __attribute__((overloadable))
{
    NSLog(@"Add NSString %@",num);
}
void add(NSNumber * num) __attribute__((overloadable))
{
    NSLog(@"Add NSNumber %@",num);
}
```

7.<font color="#FF6100" >__attribute__((objc_subclassing_restricted))</font>，在类的声明文件中加入这一行，表示这个类，不能被继承，就是java类中的final类。如果试图去继承这个类，编译会报错：
```objc
//customClass.h
__attribute__((objc_subclassing_restricted))
@interface customClass : NSObject
```

8.<font color="#FF6100" >__attribute__((objc_designated_initializer))</font>,这个用于类的初始化方法。对于对象而言，在生成一个实例的时候，如果需要提供必要的信息来完成初始化，才能使该对象正确的完成后面的工作，这种初始化方法叫"全能初始化方法"(designated initializer)。如果创建一个类的方法有多种，相应的，这个类便会有多个初始化方法。如果这些初始化分别初始化类中不同的属性时，那么在实际使用的过程中，当一使用同一个对象，做同一件事的时候，创建对象使用的是不同的初始化方法，那么问题会视后面具体的事出现不同的问题，有些可能不会造成很大的影响，有些可能会造成重大灾难。在《Effectvie objective-c 2.0》，条款16当中有详细的说明。具体的做法是指定一个全能初始化方法，让其他初始化方法去调用这个全能初始化方法。使用objc_designated_initializer还要注意，当存在继承的时候，子类在初始化时要调用父类的全能初始化方法，完成初始化，否则编译器会给出警告信息。
```objc
//Rectangle.h
@interface Rectangle : NSObject
@property (nonatomic, assign, readonly) float width;
@property (nonatomic, assign, readonly) float heigth;
- (instancetype)initWithWidth:(float) width heigth:(float) heigth __attribute__((objc_designated_initializer));
- (instancetype)init ;
@end
//Rectangle.m
@implementation Rectangle
- (instancetype)initWithWidth:(float) width heigth:(float) heigth
{
    if (self = [super init]) {
        _width = width;
        _heigth = heigth;
    }
    return self;
}
- (instancetype)init
{
    return [self initWithWidth:5.0f heigth:5.0f];
}
@end
```

9.<font color="#FF6100" >__attribute__((objc_requires_super))</font>，这个指令表示在子类重写方法时，必须在调用父类中的方法，不然后会给出警告。
```objc
//Base.h
@interface Base : NSObject
- (void)method1 __attribute__((objc_requires_super));
@end
//Base.m
@implementation Base
- (void)method1
{
    NSLog(@"Base");
}
@end
//Derived.h
@interface Derived : NSObject

@end
//Derived.m
@implementation Derived
- (void)method1
{
    [super method1];
    NSLog(@"Derived");
}
@end
```

10.<font color="#FF6100" >__attribute__((nonnull))</font>，这个表示函数参数不为空，由编译器检查。
```objc
- (void)testNonnull:(id)avg1 avg2:(id)avg2 __attribute__((nonnull))
{
    //do something
}
//某个作用域内
{
    [self testNonnull:nil avg2:nil];//这样使用时编译器会给出警告
}
```

[__attribute__ 总结](http://www.jianshu.com/p/29eb7b5c8b2d)

[Clang Attributes 黑魔法小记](http://blog.sunnyxx.com/2016/05/14/clang-attributes/)

[__attribute__](http://nshipster.com/__attribute__/)

[__attribute__指令](http://www.aopod.com/2016/08/03/attribute-directives/)

[Clang 3.8 documentation](http://www.jianshu.com/p/0237c34158f0)

--EOF--


