---
layout: post
title:  "Animation"
date:   2016-03-25
excerpt: "CALayer 基础知识"
tag:
- iOS 
- CALayer 
- 动画 
- CoreAnimation 
comments: true
---

## 一、iOS动画

iOS中实现一个动画十分简单，在view层面上通过调用

```
[UIView animateWithDuration:duration animations:^{
        //执行动画
 }]
```

但是它不能控制动画的暂停和组合，所以就需要用到CoreAnimation了。
iOS中的动画主要分为：基础动画（CABasicAnimation）、关键帧动画（CAKeyFrameAnimation）、动画组（CAAnimationGroup）、转场动画（CATransition），关系图如下

![动画关系图](http://upload-images.jianshu.io/upload_images/1638754-321b6d8124c991dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* CAAnimation：核心动画的基础类，不能直接使用，负责动画运行时间、速度控制，本身是实现了CAMediaTiming协议
* CAPropertyAnimation：属性动画的基类，即通过属性进行动画设置，不能直接使用。
* CAAnimationGroup：动画组，可以通过动画组来进行所有动画行为的统一控制，组中所有动画可以并发执行。
* CATransition：转场动画，主要通过滤镜进行动画效果设置。
* CABasicAnimation：基础动画，通过修改属性进行动画参数控制，只有初始状态和结束状态。
* CAKeyFrameAnimation：关键帧懂的规划，通过修改属性进行动画，可以有多个状态控制。

## 二、简单动画的实现（CABasicAnimation）

>You can perform simple animations implicitly or explicitly depending on your needs. Implicit animations use the default timing and animation properties to perform an animation, whereas explicit animations require you to configure those properties yourself using an animation object. So implicit animations are perfect for situations where you want to make a change without a lot of code and the default timing works well for you.

apple提供隐式动画和显式动画两种，隐式动画使用系统默认的时间和属性，如果要显式动画，需要用户自己配置属性。当用户想要用少量代码和默认的时间实现简单的动画，用隐式就行了。

> **什么是隐式动画，显式动画？**
> 
* 隐式动画就是不需要手动调用动画方法，系统默认提供的动画效果，时间是1/4秒，root layer没有隐式动画。当你改变了layer的属性，apple会立刻把这个属性改变，但是会通过呈现树以动画的形式展示改变的效果（这也就是为什么apple建议通过presentationLayer的属性来访问当前屏幕上的layer信息了），这个动态改变的过程就是隐式动画。
* 显式动画是开发者自定义的动画效果，可以设置属性改变的时间长度，改变的效果（淡入淡出等）。

### 2.1 单个动画

实现简单动画通常都用`CABasicAnimation`对象和用户配置好的动画参数来实现，通过调用basicAni的`addAnimation:forKey:`方法将动画添加到layer里面,这个方法会根据填入的key值来决定让哪个属性进行动画。

> *图层动画的本质就是将图层内部的内容转化为位图经硬件操作形成一种动画效果，其实图层本身并没有任何的变化* 
**也就是说动画不会从根本上改变layer的属性，只是展示属性变化的效果**

简单动画会在一定时间内通过动画的形式改变layer的属性值，比如用动画效果将透明度opacity从1到0：
以透明度为例实现简单动画：

```
CABasicAnimation *fade = [CABasicAnimation animationWithKeyPath:@"opacity"];
fade.fromValue = [NSNumber numberWithFloat:1.0];
fade.toValue = [NSNumber NumberWithFloat:0.0];
fade.duration = 1.0;
//注意key相当于给动画进行命名，以后获得该动画时可以使用此名称获取
[self.layer addAnimation:fade forKey:@"opacity];
//动画不会真正改变opacity的值，在动画结束后要手动更改
self.opacity = 0;
```

还可以通过animation 的delegate方法监测动画是否执行完毕

```
CABasicAnimation *fade = [CABasicAnimation animationWithKeyPath:@"opacity"];
fade.fromValue = [NSNumber numberWithFloat:1.0];
fade.toValue = [NSNumber NumberWithFloat:0.0];
fade.duration = 1.0;
fade.delegate = self;
//注意key相当于给动画进行命名，以后获得该动画时可以使用此名称获取
[self.layer addAnimation:fade forKey:@"opacity];
//动画不会真正改变opacity的值，在动画结束后要手动更改

//代理方法
- (void)animationDidStart:(CAAnimation *)anim{
    NSLog(@"START");
}

- (void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag{
    //如果不通过动画事务将隐式动画关闭，会出现动画运行完毕后会从起点突变到终点。
    [CATransaction begin];
    [CATransaction setDisableActions:YES];
    self.layer.position = [[anim valueForKey:@"KPOINT"] CGPointValue];
    [CATransaction commit];
}
```

![不更改透明度](http://upload-images.jianshu.io/upload_images/1638754-931ede35df410343.gif?imageMogr2/auto-orient/strip)



![更改了透明度](http://upload-images.jianshu.io/upload_images/1638754-fe9af10232a487db.gif?imageMogr2/auto-orient/strip)


>Tip: When creating an explicit animation, it is recommended that you always assign a value to the fromValue property of the animation object. If you do not specify a value for this property, Core animation uses the layer’s current value as the starting value. If you already updated the property to its final value, that might not yield the results you want.

显式动画，apple建议你总是给`fromValue`赋值，如果你不赋值给`fromValue`，系统会取当前的属性值作为起始点，如果你更新了最终值，可能会得到意向不到的结果(不是很明白它的意思,我的理解是*如果没更改最终值，这个属性是不会被动画改变的，动画只是视觉上的效果，原因见下*)。

>Unlike an implicit animation, which updates the layer object’s data value, an explicit animation does not modify the data in the layer tree. Explicit animations only produce the animations. At the end of the animation, Core Animation removes the animation object from the layer and redraws the layer using its current data values.
隐式动画会直接改变该属性的值，而显式动画只是一个动画效果，Core Animation会根据属性的最终值重回layer，所有必须在动画结束之后手动更改属性值，除非你不想改变该属性。

### 2.2 多个动画组合

实现移动过程中旋转的动画效果：

```
-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    CGPoint p = [[touches anyObject] locationInView:self.view];
    CALayer *layer = self.leftBottomView.layer;
    self.layer = layer;
    //添加动画
    [self positionLayer:layer position:p];
    [self rotate:layer];
}

-(void)rotate:(CALayer *)layer{
    CABasicAnimation *basicAnimation=[CABasicAnimation animationWithKeyPath:@"transform.rotation.z"];
    basicAnimation.toValue=[NSNumber numberWithFloat:M_PI_2*3];
    basicAnimation.duration=3.0;
    //添加动画到图层，注意key相当于给动画进行命名，以后获得该动画时可以使用此名称获取
    [layer addAnimation:basicAnimation forKey:@"KCBasicAnimation_Rotation"];
}
-(void)positionLayer:(CALayer *)layer position:(CGPoint)p{
    CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:@"position"];
    animation.toValue = [NSValue valueWithCGPoint:p];
    animation.duration = 3;
    animation.delegate = self;
    [animation setValue:[NSValue valueWithCGPoint:p] forKey:@"KPOINT"];
    [layer addAnimation:animation forKey:@"KPOSITION"];
}

- (void)animationDidStart:(CAAnimation *)anim{
    NSLog(@"START");
    [self.layer animationForKey:@"KPOSITION"];
}

- (void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag{
    [CATransaction begin];
    [CATransaction setDisableActions:YES];
    self.layer.position = [[anim valueForKey:@"KPOINT"] CGPointValue];
    [CATransaction commit];
}
```

![多个动画结合](http://upload-images.jianshu.io/upload_images/1638754-09eaf02c496d29be.gif?imageMogr2/auto-orient/strip)

## 三、关键帧动画（CAKeyFramedAnimation）

### 3.1 关键帧动画简单实现

CABasicAnimation只能设定初始和最终值，动画也只能是简单的从一个值到另一个值的变化。CAKeyFramedAnimation能够通过改变多个值，让你们的动画能够以线性或者非线性的方式展现。可以把keyFramedAnimation看成是许多个basicAnimation组合而成，每两个变化值之间的动画看成一个简单动画。

以改变layer的position为例通过设置关键帧，做出曲线动画，它会根据设定的`path`值
![Uploading keyframe_717706.gif . . .]（`path`是`CGPathRef`类型），通过描绘路径进行关键帧动画控制：

```
-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    CGPoint p = [[touches anyObject] locationInView:self.view];
    CALayer *layer = self.leftBottomView.layer;
    self.layer = layer;
    //开始动画
    [self keyFramed:layer];
    //动画结束之后改变layer位置
    layer.position = CGPointMake(layer.position.x, layer.position.y+200);
}

-(void)keyFramed:(CALayer *)layer{
    CAKeyframeAnimation *keyFramedAnimation = [CAKeyframeAnimation animationWithKeyPath:@"position"];
    //创建路径
    CGMutablePathRef path = CGPathCreateMutable();
    CGFloat x = layer.position.x;
    CGFloat y = layer.position.y;
    //添加贝塞尔曲线路径
    CGPathMoveToPoint(path, nil, x, y);
    CGPathAddCurveToPoint(path, nil, x+200, y, x+200, y+100, x, y+100);
    CGPathAddCurveToPoint(path, nil, x+200, y+100, x+200, y+200, x, y+200);
    keyFramedAnimation.path = path;
    keyFramedAnimation.duration = 5;
    [layer addAnimation:keyFramedAnimation forKey:@"KEYFRAME"];
}
```

![keyFrameAnimation](http://upload-images.jianshu.io/upload_images/1638754-67a898672bc5a590.gif?imageMogr2/auto-orient/strip)

### 3.2 其他属性解析

* calculationMode：
>The calculationMode property defines the algorithm to use in calculating the animation timing. The value of this property affects how the other timing-related properties are used

`calculationMode`，动画计算模式，规定了动画时间的算法，对与时间有关的动画属性起作用。它是一个字符串类型，有如下：

```
//线性动画，是计算模式的默认值
kCAAnimationLinear
//离散动画，每帧之间没有过渡
kCAAnimationDiscrete
//均匀动画，会忽略keyTimes
kCAAnimationPaced
//平滑执行
kCAAnimationCubic
//平滑均匀执行
kCAAnimationCubicPaced
```

各个值的动画效果示意图：

![calculationMode示意图](http://upload-images.jianshu.io/upload_images/1638754-b579730725d61be2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* keyTimes:
> The keyTimes property specifies time markers at which to apply each keyframe value

`keyTimes`用于设置每帧之间的动画执行时间，但是只有在`calculationMode`为`KCAAnimationLinear` 、`KCAAnimationDiscrete `、`KCAAnimationCubic`的时候才有效。

改变keyTimes来改变每帧的动画时间：

```
CAKeyframeAnimation *keyFramedAnimation = [CAKeyframeAnimation animationWithKeyPath:@"position"];
    CGMutablePathRef path = CGPathCreateMutable();
    CGFloat x = layer.position.x;
    CGFloat y = layer.position.y;
    CGPathMoveToPoint(path, nil, x, y);
    CGPathAddCurveToPoint(path, nil, x+200, y, x+200, y+100, x, y+100);
    CGPathAddCurveToPoint(path, nil, x+200, y+100, x+200, y+200, x, y+200);
    keyFramedAnimation.path = path;
    keyFramedAnimation.duration = 5;
    keyFramedAnimation.calculationMode = kCAAnimationCubicPaced;
    keyFramedAnimation.keyTimes = @[@0.0,@0.2,@1.0];
    [layer addAnimation:keyFramedAnimation forKey:@"KEYFRAME"];
```

![keytime](http://upload-images.jianshu.io/upload_images/1638754-3151f30c49bcd62b.gif?imageMogr2/auto-orient/strip)

## 四、动画组

CABasicAnimation和CAKeyFramedAnimatio一次只能改变一个属性，显示开发中要实现的动画效果一般都是多个动画一起执行的，比如平移过程中加入旋转效果等等。通过CAAnimationGroup能让开发者自由组合多个动画效果。

动画组的实现并不复杂，只需要将各个动画创建好再添加到动画组内就行了。

来给上面的关键帧动画加上旋转效果试试：

```
-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    CGPoint p = [[touches anyObject] locationInView:self.view];
    CALayer *layer = self.leftBottomView.layer;

    CAAnimationGroup *group = [CAAnimationGroup animation];
    CAKeyframeAnimation *key = [self createKeyFramed:layer];
    CABasicAnimation *rotate = [self createRotate:layer];
    group.animations = @[key,rotate];
    [layer addAnimation:group forKey:@"GROUP"];

    layer.position = CGPointMake(layer.position.x, layer.position.y+200);
}


-(CAKeyframeAnimation *)createKeyFramed:(CALayer *)layer{
    CGFloat x = layer.position.x;
    CGFloat y = layer.position.y;

    CGMutablePathRef path = CGPathCreateMutable();
    CGPathMoveToPoint(path, nil, x, y);
    CGPathAddCurveToPoint(path, nil, x+200, y, x+200, y+100, x, y+100);
    CGPathAddCurveToPoint(path, nil, x+200, y+100, x+200, y+200, x, y+200);
   
    CAKeyframeAnimation *keyFramedAnimation = [CAKeyframeAnimation animationWithKeyPath:@"position"];
    keyFramedAnimation.path = path;
    keyFramedAnimation.duration = 5;
    [layer addAnimation:keyFramedAnimation forKey:@"KEYFRAME"];
    return keyFramedAnimation;
}

-(CABasicAnimation *)createRotate:(CALayer *)layer{
    CABasicAnimation *rotate=[CABasicAnimation animationWithKeyPath:@"transform.rotation.z"];
    rotate.toValue =[NSNumber numberWithFloat:M_PI_2*3];
    rotate.duration =5.0;
    [layer addAnimation:rotate forKey:@"KCBasicAnimation_Rotation"];
    return rotate;
}
```

![GROUP](http://upload-images.jianshu.io/upload_images/1638754-5749709e018243aa.gif?imageMogr2/auto-orient/strip)

官方文档给的示例代码：

```
// Animation 1
CAKeyframeAnimation* widthAnim = [CAKeyframeAnimation animationWithKeyPath:@"borderWidth"];
NSArray* widthValues = [NSArray arrayWithObjects:@1.0, @10.0, @5.0, @30.0, @0.5, @15.0, @2.0, @50.0, @0.0, nil];
widthAnim.values = widthValues;
widthAnim.calculationMode = kCAAnimationPaced;
 
// Animation 2
CAKeyframeAnimation* colorAnim = [CAKeyframeAnimation animationWithKeyPath:@"borderColor"];
NSArray* colorValues = [NSArray arrayWithObjects:(id)[UIColor greenColor].CGColor,
            (id)[UIColor redColor].CGColor, (id)[UIColor blueColor].CGColor,  nil];
colorAnim.values = colorValues;
colorAnim.calculationMode = kCAAnimationPaced;
 
// Animation group
CAAnimationGroup* group = [CAAnimationGroup animation];
group.animations = [NSArray arrayWithObjects:colorAnim, widthAnim, nil];
group.duration = 5.0;
 
[myLayer addAnimation:group forKey:@"BorderChanges"];
```

## 五、转场动画（CATransition）

转场动画会为layer的转换添加一个视觉效果，最常见的是一个layer的消失和另一个layer的出现。


![4196_141022104125_1.jpg](http://upload-images.jianshu.io/upload_images/1638754-e0dbedc83859a81a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

子类型：

![4196_141022104212_1.jpg](http://upload-images.jianshu.io/upload_images/1638754-a35008479094edc9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
CATransition* transition = [CATransition animation];
transition.startProgress = 0;
transition.endProgress = 1.0;
transition.type = kCATransitionPush;
transition.subtype = kCATransitionFromRight;
transition.duration = 1.0;
 
// Add the transition animation to both layers
[myView1.layer addAnimation:transition forKey:@"transition"];
[myView2.layer addAnimation:transition forKey:@"transition"];
 
// Finally, change the visibility of the layers.
myView1.hidden = YES;
myView2.hidden = NO;
```

## 六、暂停和继续动画

在iOS动画时间是动画十分重要的一部分，Core Animation 通过调用 `CAMediaTiming`协议的属性和方法来精确的设定时间信息。

> However, the local time of a layer can be modified by its parent layers or by its own timing parameters. For example, changing the layer’s speed property causes the duration of animations on that layer (and its sublayers) to change proportionally

每个动画都有自己的local time，用来管理动画时间信息。通常两个layer的local time是十分接近的，因此开发者不必去关心这方面的事情，但是改变了layer的`speed`属性的话就会导致动画的时间周期改变，从而影响各个layer动画local time。

beginTime指定了动画开始之前的的延迟时间。这里的延迟从动画添加到可见图层的那一刻开始测量，默认是0（就是说动画会立刻执行）。

speed是一个时间的倍数，默认1.0，减少它会减慢图层/动画的时间，增加它会加快速度。如果2.0的速度，那么对于一个duration为1的动画，实际上在0.5秒的时候就已经完成了。

timeOffset和beginTime类似，但是和增加beginTime导致的延迟动画不同，增加timeOffset只是让动画快进到某一点，例如，对于一个持续1秒的动画来说，设置timeOffset为0.5意味着动画将从一半的地方开始。

和beginTime不同的是，timeOffset并不受speed的影响。所以如果你把speed设为2.0，把timeOffset设置为0.5，那么你的动画将从动画最后结束的地方开始，因为1秒的动画实际上被缩短到了0.5秒。然而即使使用了timeOffset让动画从结束的地方开始，它仍然播放了一个完整的时长，这个动画仅仅是循环了一圈，然后从头开始播放。

Core Animation的 *马赫时间* ，可以使用`CACurrentMediaTime`来访问，它返回了设备自从上次启动到现在的时间（然而这并没什么卵用），它的真实作用在于对动画的时间测量提供了一个相对值。

每个CALayer和CoreAnimation都有自己的本地时间概念，系统提供了两个方法：

```
-(CFTimeInterval)convertTime:(CGTimeInterval)t fromLayer:(CALayer *)l;
-(CFTimeInterval)convertTime:(CGTimeInterval)t toLayer:(CALayer *)l;
```

配合父图层/动画图层关系中的`beginTime`，`timeOff`,`speed`，可以控制动画的暂停，快退/快进的功能：

用一个小demo实现暂停、继续动画功能：

```
- (void)pauseAnimation:(CALayer *)layer{
    CFTimeInterval pauseTime = [self.layer convertTime:CACurrentMediaTime() fromLayer:nil];
    layer.timeOffset = pauseTime;
    layer.speed = 0;
}

- (void)resumeAnimation:(CALayer *)layer{
    CFTimeInterval pauseTime = [self.layer timeOffset];
    layer.timeOffset = 0;
    layer.beginTime = 0;
    layer.speed = 1;
    CFTimeInterval timeSincePause = [self.layer convertTime:CACurrentMediaTime() fromLayer:nil] - pauseTime;
    layer.beginTime = timeSincePause;
}

- (IBAction)pause:(id)sender {
    self.stop = !self.stop;
    if (self.stop) {
        [self animationResume];
    }else{
        [self animationPause];
    }
}
```

得到的效果
![pause](http://upload-images.jianshu.io/upload_images/1638754-753dd3a92d8c9602.gif?imageMogr2/auto-orient/strip)



## 七、动画事务（CATransaction）

CATransaction负责管理动画、动画组的创建和执行。大部分情况下开发者不需要手动创建trasaction（动画事务），但如果要更精确的控制动画的属性，比如`duration`，timing function等，就要通过手动创建CATrasaction。

开启新的动画事务调用`start`方法，结束事务调用`commit`方法。

apple给的简单示例代码,通过：

```
[CATransaction begin];
theLayer.zPosition=200.0;
theLayer.opacity=0.0;
[CATransaction commit];
```

CATransaction通过block运行开发者在动画在动画结束之后执行一些其他操作：

```
[CATransaction setCompletionBlock:^{
          //执行动画结束之后的操作
    }];
```

CATransaction通过`setValue:forKey`来改变动画的属性,key值是使用的是系统提供的字符串类型值，以改变动画时间为例：

```
[CATransaction begin];
[CATransaction setValue:@10 forKey:kCATransactionAnimationDuration];
[CATransaction comment];
```

除了时间外还有：

```
kCATransactionDisableActions
kCATransactionAnimationTimingFunction
kCATransactionCompletionBlock
```

CATransaction允许一个事务内嵌套多个事务的操作，但是每个动画事务的`begin`和`commit`必须配套使用。

```
//outer transaction
[CATransaction begin];
CATransaction setValue:@2 forKey:kCATransactionAnimationDuration];
layer.positon = CGPointMake(0,0);

//inner transaction
[CATransaction begin];
[CATranscation setValue:@3 forKey:kCATransactionAnimationDuration];
layer.zPosition = 200;
layer.opacity = 0;
//inner transaction completed
[CATransaction commit];

//outer transaction completed
[CATranscation commit]
```