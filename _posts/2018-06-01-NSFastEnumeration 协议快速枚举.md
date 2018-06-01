---
layout:     post
title:      NSFastEnumeration协议快速枚举
date:       2018-06-01
summary:    代码中经常能见到的for-in语句，背后的实现细节。
categories: objectvie-c NSFastEnumeration
---

标签： objectvie-c NSFastEnumeration


# 问题起源

平时写代码的时候，当遍历一个数组的时候免不了要使用for语句，其中for-in语句经常被使用到，**其具有高效、语法简洁、遍历的时候不能修改元素的值的特点**，而并没有去关心它与for语句的区别，更没关心它是如何实现的，一直以来，都认为只有在系统提供的集合类型中才能使用for-in语句。直到今天，在学习ReactiveCocoa源码的时候，在RACSequence类中发现了，一段代码，让我思考良久，确切的说是RACSequence的子类RACArraySequence。

```
- (id)foldLeftWithStart:(id)start reduce:(id (^)(id, id))reduce {
	NSCParameterAssert(reduce != NULL);
	if (self.head == nil) return start;
	for (id value in self) {
        NSLog(@"for-in stemtent: %@",value);
		start = reduce(start, value);
	}
	return start;
}
```
我一开始只是试一试RACSequenece的用法，如：

```
NSArray *array = @[@12,@34,@12,@13,@14,@15,@16,@17,@18,@19,@20,@21,@22,@23,@24,
                       @17,@18,@19,@20,@21,@22,@23];
BOOL result = [array.rac_sequence all:^BOOL(id value) {
    return [value integerValue] < 24;
}];
```

然后就是顺着调用顺序看源码发现了在``foldLeftWithStart: reduce:``中有一个for-in语句，而且遍历的对象竟然是自己**RACArraySequence**。这个类的继承关系为：

**RACArraySequence->RACSequence->RACStream->NSObject**
其本质是一个NSObject的子类。多方打探后，发现``RACArraySequence``实现了``NSFastEnumeration``协议，让其具备了for-in的功能。好了，问题发现了，下面开始正文。

# NSFastEnumeration

这个协议很简单，可能是我见过系统框架里最简单的协议了吧。里面只有一个结构体，和一个方法。

```
typedef struct {
    unsigned long state;
    id __unsafe_unretained _Nullable * _Nullable itemsPtr;
    unsigned long * _Nullable mutationsPtr;
    unsigned long extra[5];
} NSFastEnumerationState;

- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state objects:(id __unsafe_unretained _Nullable [_Nonnull])buffer count:(NSUInteger)len;
```
只要实现这个协议，任何类都具备使用for-in能力。网上有一很多资料解释这个结构和这个方法的作用，有一些不是很准确。对这个方法最重要的是**它会调用n次，n=array.length / 16。准确来说，这个方法会分段生成数组，这个数组就是参数buffer。**

1.	NSFastEnumerationState只用到了两个字段：itemPtr和mutationPtr。前者指向buffer的首地址，后者用于检测在遍历的过程中是否修改元素，是则抛出异常。
2. 参数buffer为系统提供的缓冲数组，参数len为缓冲数组的最大长度。

对``foldLeftWithStart: reduce:``方法断点调试会发现，``countByEnumeratingWithState: objects: count:``方法会调用三次。

```
- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state objects:(__unsafe_unretained id[])stackbuf count:(NSUInteger)len {
	NSCParameterAssert(len > 0);
	//上一次生成的数组时state->state保存的index等于源数组的长度时，表示数组生成完毕，返回0
	if (state->state >= self.backingArray.count) {
		// Enumeration has completed.
		return 0;
	}
	//方法第一次调用时state->state是为0的，对state结构体进行初始化设置
	if (state->state == 0) {
		state->state = self.offset;
		// 因为遍历期间不能修改元素的值，所以mutationsPtr要设置为非空
		state->mutationsPtr = state->extra;
	}

	state->itemsPtr = stackbuf;

	NSUInteger startIndex = state->state;
	NSUInteger index = 0;

	for (id value in self.backingArray) {
		// Constructing an index set for -enumerateObjectsAtIndexes: can actually be
		// slower than just skipping the items we don't care about.
		if (index < startIndex) {
			++index;
			continue;
		}

		stackbuf[index - startIndex] = value;

		++index;
		if (index - startIndex >= len) break;
	}

	NSCAssert(index > startIndex, @"Final index (%lu) should be greater than start index (%lu)", (unsigned long)index, (unsigned long)startIndex);
	//记录本次生成的数组的长度，以后每一次生成数组时，其下标为相对上一次生成的数组的长度
	//比如：源数组长度为22，则第一次生成的数组stackbuf长度为16，stage->state= 16，下次生成的数组为长度则为6，index则在16的基础上偏移
	state->state = index;
	//返回本次生成的stackbuf数组的实际长度
	return index - startIndex;
}
```
反编译以下例子：

```
int main() {
    NSArray *arr = @[@12,@13,@14,@15,@16];
    for (id value in arr) {
        printf("%ld\n",[value integerValue]);
    }
    return 0;
}
```

汇编结果：

```
memset(var_108, 0x0, 0x40);
rax = [var_C0 retain];
var_160 = rax;
rax = [rax countByEnumeratingWithState:var_108 objects:var_B0 count:0x10];
var_168 = rax;
if (rax != 0x0) {
        var_170 = *var_F8;
        var_178 = var_108 + 0x10;
        var_180 = 0x0;
        var_188 = var_168;
        do {
                do {
                        var_190 = var_188;
                        var_198 = var_180;
                        if (**var_178 != var_170) {
                                objc_enumerationMutation(var_160);
                        }
                        printf("%ld\n", [*(var_100 + var_198 * 0x8) integerValue]);
                        var_188 = var_190;
                        var_180 = var_198 + 0x1;
                } while (var_198 + 0x1 < var_190);
                rax = [var_160 countByEnumeratingWithState:var_108 objects:var_B0 count:0x10];
                var_180 = 0x0;
                var_188 = rax;
        } while (rax != 0x0);
}

```

