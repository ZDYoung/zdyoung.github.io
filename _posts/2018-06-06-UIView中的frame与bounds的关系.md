---
layout:     post
title:      UIView中的frame与bounds的关系
date:       2018-06-06
summary:    frame与bounds之间的关系，以及使用bounds与center的效果等同于frame
categories: objectvie-c UIView
---

标签： objectvie-c UIView

# 概念

在UIView中，frame和bounds属性都是CGRect类型，两者非常相似，在平时的开发中，如果使用AutoLayout进行布局那么frame基本上用不上，而bounds使用的场景也不多，但是在用到的时候，经常想不明白，不明所以，这次要彻底搞懂。

frame是相对于父视图的坐标的，frame中的x和y，代表视图在父视图的起始坐标；frame中的size，代表视图在父视图中的大小。bounds是相对于视图本身而言，bounds的x和y定义了视图左上角的点，当有子视图的时候，子视图以父视图的左上角为原点(0,0)来决定自身在父视图的位置。所以父视图的bounds影响的是其子视图在父视图上的位置，视图的bounds属性的x和y默认为0，即以视图的左上角的点为坐标原点，大小为视图的大小。

*	frame：描述当前视图在其父视图中的位置和大小。
*	bounds：描述当前视图在其自身坐标系统中的位置和大小。
*	center：描述当前视图的中心点在其父视图中的位置。

想要确定视图在父视图上的大小和位置，可以通过：

1.	设置frame。

2.	设置bounds和center，bounds可以确定视图大小，center可以确定视图位置。

frame、bounds、center的关系：

```
center.x = frame.origin.x + frame.size.width / 2
center.y = frame.origin.y + frame.size.heigth / 2
```

当设置bounds不设置frame时：

```
frame.origin.x = -bounds.size.width / 2
frame.origin.y = -bounds.size.heigth / 2

frame.size.width = bounds.size.width
frame.size.width = bounds.size.width
```

因此，当设置bounds不设置frame时，其center始终为0。

# Demo

网上看到一个例子很详细：

```
UIView *view1 = [[UIView alloc] initWithFrame:CGRectMake(20, 20, 280, 250)];  
[view1 setBounds:CGRectMake(-20, -20, 280, 250)];  
view1.backgroundColor = [UIColor redColor];  
[self.view addSubview:view1];//添加到self.view  
NSLog(@"view1 frame:%@========view1 bounds:%@",NSStringFromCGRect(view1.frame),NSStringFromCGRect(view1.bounds));  
  
UIView *view2 = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 100, 100)];  
view2.backgroundColor = [UIColor yellowColor];  
[view1 addSubview:view2];//添加到view1上,[此时view1坐标系左上角起点为(-20,-20)]  
NSLog(@"view2 frame:%@========view2 bounds:%@",NSStringFromCGRect(view2.frame),NSStringFromCGRect(view2.bounds));  
```

运行结果：

```
2018-06-06 16:44:08.794861+0800 LearnCALayer[47486:28184109] view1 frame:\{\{20, 20\}, {280, 250}}========view1 bounds:\{\{-20, -20\}, {280, 250}}
2018-06-06 16:44:08.795074+0800 LearnCALayer[47486:28184109] view2 frame:\{\{0, 0\}, {100, 100}}========view2 bounds:\{\{0, 0\}, {100, 100}}
```

![frame_eg](/images/frame_eg.jpg)