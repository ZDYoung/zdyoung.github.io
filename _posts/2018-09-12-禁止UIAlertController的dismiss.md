---
layout:     post
title:      禁止UIAlertController的dimiss
date:       2018-09-12
summary:    使用runtime hook UIAlertController的私有方法
categories: objectvie-c Runtime
---
标签： objectvie-c Runtime

UIAlertController是在iOS 8.0以后才出现的，用于弹窗提示的视图，在8.0 之前使用的是UIAlertView。UIAlertController有两种模式：Alert和Sheet。只有在Alert模式下，才可以向UIAlertController中添加UITextField，否则会崩溃。默认情况都是点击UIAlertController上的按钮之后，执行相应回调最后dismiss掉。在没有UITextField的时候，这样没有任何问题，但是在UITextField存在的情况下，且需要对输入进行合法性验证，只有输入合法时点击UIAlertController才进行回调处理。按照系统的默认方式在点击UIAlertController按钮时流程为：

1.	如果输入合法，执行处理回调；如果输入不合法，给出提示，不执行回调。

2.	UIAlertController dismiss

这里不管输入合法与否，都是会dismiss，如何做到输入不合法，不dismiss，直到输入合法才dismiss。目前能想到的只有两种方法：

1.	自己重写一个UIAlertController，好处是可以根据实际需求灵活配置，包括大小，颜色之类的，坏处是工作量比较大。

2.	从系统UIAlertController入手，根据方法调用过程hook相关方法。

在UIAlertController回调里打一个断点，然后查看函数的调用栈：
![UIAlertController](/images/UIAlertController.png)

从函数栈中猜想，系统在这个过程中应该是调用了两个关键方法。首先调用_dismissAnimated:triggeringAction:triggeredByPopoverDimmingView:  将UIAlertController dismiss，接着调用_fireOffActionOnTargetIfValidForAction:来执行回调。

验证：

```
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        //在iOS 11.0以上UIAlertController的dimiss的私有方法是_dismissAnimated:triggeringAction:triggeredByPopoverDimmingView:dismissCompletion:
        //iOS 11.0以下是_dismissAnimated:triggeringAction:triggeredByPopoverDimmingView:
        SEL originalSelectorWitCompletion = NSSelectorFromString(@"_dismissAnimated:triggeringAction:triggeredByPopoverDimmingView:dismissCompletion:");
        SEL originalSelector = NSSelectorFromString(@"_dismissAnimated:triggeringAction:triggeredByPopoverDimmingView:");
        
        SEL swizzledSelector = @selector(def_dismissAnimated:triggeringAction:triggeredByPopoverDimmingView:);
        
        SEL swizzledSelectorWithCompletion = @selector(def_dismissAnimated:triggeringAction:triggeredByPopoverDimmingView:dismissCompletion:);
        
        Method originalMethodWithCompletion = class_getInstanceMethod(class, originalSelectorWitCompletion);
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        Method swizzledMethodWithCompletion = class_getInstanceMethod(class, swizzledSelectorWithCompletion);
        if (originalMethodWithCompletion) {
            method_exchangeImplementations(originalMethodWithCompletion, swizzledMethodWithCompletion);
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

- (void)def_dismissAnimated:(BOOL)animation triggeringAction:(UIAlertAction *)action triggeredByPopoverDimmingView:(id)view dismissCompletion:(id)handler
{
    if (action.style == UIAlertActionStyleCancel) {
        [self def_dismissAnimated:animation triggeringAction:action triggeredByPopoverDimmingView:view dismissCompletion:handler];
    } else {
        if (self.validateBlock && self.textFields.count) {
            self.isDismiss = self.validateBlock(self.textFields.firstObject.text);
            if (self.isDismiss) {
                [self def_dismissAnimated:animation triggeringAction:action triggeredByPopoverDimmingView:view dismissCompletion:handler];
            }
        } else {
            [self def_dismissAnimated:animation triggeringAction:action triggeredByPopoverDimmingView:view dismissCompletion:handler];
        }
    }
}

- (void)def_dismissAnimated:(BOOL)animation triggeringAction:(UIAlertAction *)action triggeredByPopoverDimmingView:(id)view
{
    if (action.style == UIAlertActionStyleCancel) {
        [self def_dismissAnimated:animation triggeringAction:action triggeredByPopoverDimmingView:view];
    } else {
        if (self.validateBlock && self.textFields.count) {
            self.isDismiss = self.validateBlock(self.textFields.firstObject.text);
            if (self.isDismiss) {
                [self def_dismissAnimated:animation triggeringAction:action triggeredByPopoverDimmingView:view];
            }
        } else {
            [self def_dismissAnimated:animation triggeringAction:action triggeredByPopoverDimmingView:view];
        }
    }
}

- (void)setIsDismiss:(BOOL)isDismiss
{
    objc_setAssociatedObject(self, @selector(isDismiss), @(isDismiss), OBJC_ASSOCIATION_ASSIGN);
}

- (BOOL)isDismiss
{
    return [(NSNumber *)objc_getAssociatedObject(self, _cmd) boolValue];
}

- (void)setValidateBlock:(DEFAlertControllerTextFieldValidateBlock)validateBlock
{
    objc_setAssociatedObject(self, @selector(validateBlock), validateBlock, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (DEFAlertControllerTextFieldValidateBlock)validateBlock
{
    return objc_getAssociatedObject(self, @selector(validateBlock));
}
```

实际结果表明，上述分析是正确的。有一点需要注意的是，在iOS 11.0以后UIAlertController使用的私有API方法名和11.0以下使用的不一样。