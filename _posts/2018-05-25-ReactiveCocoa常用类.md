---
layout:     post
title:      ReactiveCocoa
date:       2018-05-25
summary:    一些常用类、方法和注意事项。
categories: objectvie-c ReactiveCocoa
---

标签： objectvie-c ReactiveCocoa

# ReactiveCocoa常用类
------

* RACSingnal
* RACSubject
* RACCommand
* RACMulticastConnection
* RACChannelTerminal

## RACSingal相关的类

<!-- ReactiveCocoa里信号都是泛型的，如UITextField的信号是NSString类型的信号，UISlider的信号是NSNumber信号，在涉及到信号的绑定时，不同类型的信号需要map一下。之前我将UITextField信号绑定在UISlider上，想在UITextField上输入数字的时候UISlider的value值跟着变，虽然可以用RACChannelTerminal来实现，但我就是想用RAC宏来绑定信号。刚开始是这样的： -->
```
```

```
RACEmptySignal ：空信号，用来实现 RACSignal 的 +empty 方法；
RACReturnSignal ：一元信号，用来实现 RACSignal 的 +return:方法；
RACDynamicSignal ：动态信号，使用一个 block - 来实现订阅行为，我们在使用 RACSignal 的 +createSignal: 方法时创建的就是该类的实例；
RACErrorSignal ：错误信号，用来实现 RACSignal 的 +error: 方法；
RACChannelTerminal ：通道终端，代表 RACChannel 的一个终端，用来实现双向绑定。
```
RACSignal代表冷信号，与订阅者是一对一的关系，当有不同的订阅者时，消息都会完整的重发。热号信不管有没有订阅者，都会发送消息。

冷信号：

```
RACSignal *signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:@1];
    [subscriber sendNext:@2];
    [subscriber sendNext:@3];
    [subscriber sendCompleted];
    return nil;
}];
NSLog(@"Signal was created.");
[[RACScheduler mainThreadScheduler] afterDelay:0.1 schedule:^{
    [signal subscribeNext:^(id x) {
        NSLog(@"Subscriber 1 recveive: %@", x);
    }];
}];

[[RACScheduler mainThreadScheduler] afterDelay:1 schedule:^{
    [signal subscribeNext:^(id x) {
        NSLog(@"Subscriber 2 recveive: %@", x);
    }];
}];
```
两个订阅者都会收到1，2，3，但两个订阅者收到的不是同一个消息，而是两次不同的消息。

热信号：

```
RACMulticastConnection *connection = [[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [[RACScheduler mainThreadScheduler] afterDelay:1 schedule:^{
        [subscriber sendNext:@1];
    }];
    
    [[RACScheduler mainThreadScheduler] afterDelay:2 schedule:^{
        [subscriber sendNext:@2];
    }];
    
    [[RACScheduler mainThreadScheduler] afterDelay:3 schedule:^{
        [subscriber sendNext:@3];
    }];
    
    [[RACScheduler mainThreadScheduler] afterDelay:4 schedule:^{
        [subscriber sendCompleted];
    }];
    return nil;
}] publish];
[connection connect];
RACSignal *signal = connection.signal;
    
NSLog(@"Signal was created.");
[[RACScheduler mainThreadScheduler] afterDelay:1.1 schedule:^{
    [signal subscribeNext:^(id x) {
        NSLog(@"Subscriber 1 recveive: %@", x);
    }];
}];
    
[[RACScheduler mainThreadScheduler] afterDelay:2.1 schedule:^{
    [signal subscribeNext:^(id x) {
        NSLog(@"Subscriber 2 recveive: %@", x);
    }];
}];
```
这次，这两个订阅者收到的都是都一组消息，两都共享信号发出的消息，因此结果与第一个不同。热信号signal主动发送消息，分别在第一秒，第二秒和第三秒，发送了三个消息。订阅者一，在1.1秒后订阅消息，因此只能收到消息2和消息3，错过了1秒钟之前发送的消息；订阅者二，在2.1秒后开始订阅消息，因此只能收到消息3，错过了信号在2秒钟之前发送的消息。冷信号转换成热信号通过``RACMulticastConnection``，它内部使用RACSubject订阅RACSignal，前者可以表示热信号。


## RACSubject相关的类


```
RACGroupedSignal ：分组信号，用来实现 RACSignal 的分组功能；

RACBehaviorSubject ：重演最后值的信号，当被订阅时，会向订阅者发送它最后接收到的值；

RACReplaySubject ：重演信号，保存发送过的值，当被订阅时，会向订阅者重新发送这些值。
```

RACSubject是RACSignal的子类，既能发送信号，又能订阅信号，本身及其子类都是热信号。**使用RACSubject创建的信号，必需要先订阅再发送，订阅者才能收到消息。**RACSubject可以替换代理，代理对象订阅信号，委托对象发送信号。

```
// 1.创建信号
RACSubject *subject = [RACSubject subject];
// 2.订阅信号
[subject subscribeNext:^(id x) {
    NSLog(@"第一个订阅者%@", x);
}];
[subject subscribeNext:^(id x) {
    NSLog(@"第二个订阅者%@", x);
}];
// 3.发送信号
[subject sendNext:@"1"];
```
**子类RACBehaviorSubject和RACReplaySubject都可以先发送信号，再订阅信号**

```
// 1.创建信号
RACReplaySubject *replaySubject = [RACReplaySubject subject];
// 2.发送信号
[replaySubject sendNext:@1];
[replaySubject sendNext:@2];
// 3.订阅信号
[replaySubject subscribeNext:^(id x) {
    NSLog(@"第一个订阅者接收到的数据%@",x);
}];
// 订阅信号
[replaySubject subscribeNext:^(id x) {
    NSLog(@"第一个订阅者接收到的数据%@",x);
}];
```
RACReplaySubject会把之前的值都重复的再发一次，使用场景：

*	如果一个信号每被订阅一次，就需要把之前的值重复发送一遍，使用重复提供信号类。
* 	用于缓存最新的capacity数量的值

RACBehviorSubject只会向订阅者发送最新的消息。

```
RACBehaviorSubject *subject = [RACBehaviorSubject subject];
[subject subscribeNext:^(id  _Nullable x) {
    NSLog(@"1st subject: %@",x);
}];  
[subject sendNext:@"1"];   
[subject subscribeNext:^(id  _Nullable x) {
    NSLog(@"2st subject: %@",x);
}]; 
[subject sendNext:@"2"];
[subject subscribeNext:^(id  _Nullable x) {
    NSLog(@"3st subject: %@",x);
}]; 
[subject sendNext:@"3"]; 
[subject sendCompleted];
```
最开始有一个订阅者1，它立刻收到最新消息，这时没有最新消息发出，默认为nil，RACBehviorSubject有一个用于创建包含默认值的类方法``behaviorSubjectWithDefaultValue:``。然后发送1，马上又有订阅者2，这时最新的消息为1，订阅者2此时只会收到1，订阅者1也会收到最新消息，接着发送2，此时的最新消息为2，订阅者1与订阅者2都会收到消息2。最后创建订阅者3的时候，会收到最新消息2，然后发送3的时候，订阅者1，2，3收到最新消息3。

## RACCommand

在ReactiveCocoa中RACCommand的解释为用于处理某些事件的类，比如UI相关的。因此可以将其与UIButton的点击事件进行绑定，还可以对``UIButton``的``rac_signalForControlEvents``信号的订阅进行处理。RACCommand可以对事件的处理进行包装，并将处理结果发送出来。

运用场景：点击按钮，执行一个网络请求或上传数据或下载图片等，期间只有当command执行完成，按钮才可用。

```
RACCommand *command = [[RACCommand alloc] initWithEnabled:nil signalBlock:^RACSignal * _Nonnull(id  _Nullable input) {
    return [RACSignal createSignal:^RACDisposable * _Nullable(id<RACSubscriber>  _Nonnull subscriber) {
        //模拟网络处理...
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
			//将结果发送出去
            [subscriber sendNext:@"data"];
            [subscriber sendCompleted];
        });
        return nil;
    }];
}];
//与UIButton绑定
self.button.rac_command = command;
//订阅信号，获取结果
[[command.executionSignals switchToLatest] subscribeNext:^(id  _Nullable x) {
    NSLog(@"%@", x);
}];
```
不与UIButton绑定，手动执行的时候使用``-execute:``，可以将网络请求的参数传进去。``RACCommand``有一个``executing``属性用于监听当前的command是否执行完毕，默认会发送一次，可跳过。

```
[command execute:nil];
[[command.executionSignals switchToLatest] subscribeNext:^(id  _Nullable x) {
    NSLog(@"%@", x);
}];
//以上等价于
[[command execute:nil] subscribeNext:^(id  _Nullable x) {
    NSLog(@"%@",x);
}];
[[command.executing skip:1] subscribeNext:^(NSNumber * _Nullable x) {
    if ([x boolValue]) {
        NSLog(@"command is executing");
    } else {
        NSLog(@"command executed");
    }
}];
```
## RACChannelTerminal

RACChannelTerminal是RACSignal的子类，可以实现数据流的双向绑定，如model与视图之间的绑定，某一方发生变化，另一方也跟着变。UIKit中支持RACChannelTerminal的类有：

*	UIDatePicker
* 	UIStepper
* 	UISegmentedControl
*	UITextField
*  UISwitch
*  UISlider

**注意：**获取一个RACChannelTerminal可通过``RACChannelTo``宏，也可以通过框架提供的相应属性。``self.valueTextField.rac_newTextChannel`` 这个只有在输入框输入的时候才会发出新的值，如果用代码设置的text属性，则不会。使用``RACChannelTo ``宏，只有在使用代码设置text属性时才会发出新的值，当在输入框输入时，则不会。其他控件类似。***对于控件不要使用``RACChannelTo ``宏。***

绑定view与model：

```
RACChannelTerminal *text = self.nameText.rac_newTextChannel;
RACChannelTerminal *string = RACChannelTo(self.model, text);
[text subscribe:string];
[string subscribe:text];
```
# ReactiveCocoa常用宏
------

``RAC``宏总是与一个signal绑定，当signal发每产生一个value时，都是自动执行：

```
[TARGET setValue: value ?: NIL_VALUE forKeyPath:KEYPATH];
```
更新``RAC``宏绑定的，TARGET的属性。
``RACObserve``宏，用于观察TARGET的KEYPATH属性，相当于一个KVO，产生一signal，默认会发送一次信号，常与``RAC``宏一起使用。

```
//model改变时，设置label的text属性
RAC(self.outputLabel, text) = RACObserve(self.model, name); 
```

# ReactiveCocoa常用操作方法
------

## then
用于连接两个信号，当第一个信号完成，才会连接then返回的信号。

**注意: 使用then之前的信号的值会被忽略。**

底层实现：

1.	先过滤掉之前的信号发出的值。
2. 使用concat连接then返回的信号

```
RACSignal *signalA = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    //发送请求
    NSLog(@"发送上部分的请求");
    //发送信号
    [subscriber sendNext:@"上部分数据"];
    //发送完毕
    //加上后就可以上部分发送完毕后发送下半部分信号
    [subscriber sendCompleted];
    return nil;
    
}];

RACSignal *signalB = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    //发送请求
    NSLog(@"发送下部分的请求");
    //发送信号
    [subscriber sendNext:@"下部分数据"];
    return nil;
    
}];
    
//thenSignal组合信号
//then:会忽略掉第一个信号的所有值
RACSignal *thenSignal = [signalA then:^RACSignal *{
    //返回的信号就是需要组合的信号
    return signalB;
}];

[thenSignal subscribeNext:^(id x) {
    NSLog(@"%@", x);
}];
```
```
2018-05-24 19:27:48.573128+0800 ReactiveCocoa[93483:16178302] 发送上部分的请求
2018-05-24 19:27:48.573343+0800 ReactiveCocoa[93483:16178302] 发送下部分的请求
2018-05-24 19:27:48.573457+0800 ReactiveCocoa[93483:16178302] 下部分数据
```

## concat
concat底层实现:
**注意：第一个信号必须调用``sendCompleted:``**

1.	当拼接信号被订阅，就会调用拼接信号的didSubscribe
2.	didSubscribe中会先订阅第一个源信号（signalA）
3.	会执行第一个源信号（signalA）的didSubscribe
4.	第一个源信号（signalA）didSubscribe中发送值，就会调用第一个源信号（signalA）订阅者的nextBlock,通过拼接信号的订阅者把值发送出来.
5.	第一个源信号（signalA）didSubscribe中发送完成，就会调用第一个源信号（signalA）订阅者的completedBlock,订阅第二个源信号（signalB）这时候才激活（signalB）。
6.	订阅第二个源信号（signalB）,执行第二个源信号（signalB）的didSubscribe
7.	第二个源信号（signalA）didSubscribe中发送值,就会通过拼接信号的订阅者把值发送出来.

```
RACSignal *signalA = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    //发送请求
    NSLog(@"发送上部分的请求");
    //发送信号
    [subscriber sendNext:@"上部分数据"];
    //发送完毕
    //加上后就可以上部分发送完毕后发送下半部分信号
    [subscriber sendCompleted];
    return nil;
}];
RACSignal *signalB = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    //发送请求
    NSLog(@"发送下部分的请求");
    //发送信号
    [subscriber sendNext:@"下部分数据"];
    return nil;
}];
//创建组合信号
//concat:按顺序去连接(组合)
//注意:第一个信号必须调用sendCompleted
RACSignal *concat = [signalA concat:signalB];
//订阅组合信号
[concat subscribeNext:^(id x) {
    //既能拿到A信号的值,又能拿到B信号的值
    NSLog(@"x: %@", x);
}];
```

```
2018-05-24 19:30:48.735101+0800 ReactiveCocoa[93570:16181347] 发送上部分的请求
2018-05-24 19:30:48.735292+0800 ReactiveCocoa[93570:16181347] x: 上部分数据
2018-05-24 19:30:48.735426+0800 ReactiveCocoa[93570:16181347] 发送下部分的请求
2018-05-24 19:30:48.735531+0800 ReactiveCocoa[93570:16181347] x: 下部分数据
```

## zipWith

把两个信号压缩成一个信号，只有当两个信号同时发出信号内容时，并且把两个信号的内容合并成一个元组，才会触发压缩流的next事件
底层实现:

1.	定义压缩信号，内部就会自动订阅signalA，signalB
2.	每当signalA或者signalB发出信号，就会判断signalA，signalB有没有发出个信号，有就会把最近发出的信号都包装成元组发出。元组中数据的顺序与发送顺序无关，跟组合顺序有关

```
//创建信号A
RACSubject *signalA = [RACSubject subject];
//创建信号B
RACSubject *signalB = [RACSubject subject];
//压缩成一个信号
//当一个界面多个请求时,要等所有的请求都完成才能更新UI
//打印顺序跟组合顺序有关,跟发送顺序无关
RACSignal *zipSignal = [signalB zipWith:signalA];
[zipSignal subscribeNext:^(id x) {
    NSLog(@"%@", x);
}];
[signalA sendNext:@"HMJ"];
[signalB sendNext:@"GQ"];
```
```
2018-05-24 19:35:41.495498+0800 ReactiveCocoa[93673:16185744] <RACTwoTuple: 0x604000019b50> (
    GQ,
    HMJ
)
```
## combineLatestWith

将多个信号合并起来，并且拿到各个信号的最新的值,必须每个合并的signal至少都有过一次sendNext，才会触发合并的信号。
底层实现：

1.	当组合信号被订阅，内部会自动订阅signalA，signalB,必须两个信号都发出内容，才会被触发。
2.	并且把两个信号组合成元组发出。

```
RACSignal *signalA = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:@"A"];//一次sendNext
    [subscriber sendCompleted];
    return nil;
}];
RACSignal *signalB = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    [subscriber sendNext:@"B"];//一次sendNext
    [subscriber sendCompleted];
    return nil;
}];
RACSignal *combineSignal = [signalA combineLatestWith:signalB];
[combineSignal subscribeNext:^(id x) {
    NSLog(@"%@", x);
}];
```
```
2018-05-24 19:37:48.897069+0800 ReactiveCocoa[93734:16187988] <RACTwoTuple: 0x60800000d540> (
    A,
    B
)
```
## combineLates:reduce

组合多个信号，并聚合，返回聚合后的值

登录界面：当用户名与密码都输入时，登录按钮才可点击。

```
RACSignal *combineSignal = [RACSignal combineLatest:@[self.nameText.rac_textSignal, self.passwdText.rac_textSignal] reduce:^id(NSString *account, NSString *pwd){
    //block:只要源信号发送内容就会调用,组合成新的一个值
    //聚合的值就是组合信号的内容
    return @(account.length && pwd.length);
}];
//订阅信号
//    [combineSignal subscribeNext:^(id x) {
//        self.loginBtn.enabled = [x boolValue];
//
//    }];
//等同于
RAC(self.loginBtn, enabled) = combineSignal;
```

## merge

将多个信号合并成一个信号，任何一个信号有新值时都会调用，在textField中输入或者滑动slider订阅者都会收到消息。

```
RACChannelTerminal *t = [self.slider rac_newValueChannelWithNilValue:nil];
RACSignal *tt = [self.textField rac_textSignal];
RACSignal *ttt = [t merge:tt];
[ttt subscribeNext:^(id  _Nullable x) {
    NSLog(@"%@",x);
}];
```

## filter
filter方法用于过滤信号，当满足条件时才会发送消息

```
//只有当text.length>3的时候才会订阅改消息
[[textField.rac_textSignal filter:^BOOL(NSString * text) {
   
    return text.length>3;
    
}] subscribeNext:^(id x) {
   
    NSLog(@"%@",x);
    
}];
```
## take 
take:当有多个消息发送的时候，只取前面的几条

takeLast:当有多个消息发送的时候，只取最后面的几条。**订阅者必须完成调用，即调用``sendCompleted``，才能总共有多少个信号。**

```
//在textField里输入时，只会取前面的5条消息
[[[self.textField rac_textSignal] take:5]subscribeNext:^(NSString * _Nullable x) {
    NSLog(@"X:%@",x);
}];
```
## map

map方法返回映射后的新值

```
//输出字符的个数
[[textField.rac_textSignal map:^id(NSString * value) {
    return @(value.length);
}] subscribeNext:^(NSNumber * x) {
    NSLog(@"%@",x);
}];
```

## distinctUntilChanged

当上一次的值和当前的值有明显的变化就会发出信号，否则会被忽略掉。

## ignore

忽略某些字符

```
//忽略输入框后的空字符
[[self.textField.rac_textSignal ignore:@""] subscribeNext:^(NSString * _Nullable x) {
    NSLog(@"X:%@",x);
}];
```

## interval

每间隔一段时间发送一次信号。

```
//计时
RAC(lab,text) = [[RACSignal interval:1 onScheduler:[RACScheduler mainThreadScheduler]] map:^id(NSData * value) {
        return value.description;
    }];
```

## time
```
//设置超时时间为2秒,当超过2秒还没有发送消息的时候,就不会发送了
RACSignal *signal = [[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        [[RACScheduler mainThreadScheduler] afterDelay:3 schedule:^{
            [subscriber sendNext:@"Ricky"];
            [subscriber sendCompleted];
        }];
        return nil;
    }] timeout:2 onScheduler:[RACScheduler mainThreadScheduler]];
    [signal subscribeNext:^(id x) {
    
        NSLog(@"%@",x);
    }];
```

