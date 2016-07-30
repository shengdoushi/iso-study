# pop 动画库

# 概述

项目地址: https://github.com/facebook/pop/tree/master/pop

是一个动画引擎，提供了基本的动画和诸如 spring, decay 等高级动画。是著名应用 Paper 的动画引擎。因为 pop 是一个 ios, osx, tvos 三个平台上的库，下面只针对ios平台。

当前的版本为 1.0.9,  commit为 b30efd8。


# 用法
1. 构造一个动画，共有四种类型 spring, decay, basic, 和 custom。 分别对应实现类: POPSpringAnimation, POPDecayAnimation, POPBasicAnimation 和 POPCustomAnimation. 每种动画有自己的变量，无非就是设置动画的初始状态，和动画的结束状态，以及描述动画过程中变化曲线的属性。

2. 启动动画，只要将动画添加到要执行的对象(NSObject)上: 通过 [object pop_addAnimation:anim forKey:@"key"]，之后可以通过 [object pop_animationForkey:@"key"] 来获取动画，更新内部的动画属性

3. 停止动画，将动画从对象身上移除： [object pop_removeAnimationForKey:@"key"]


动画的属性通过 POPAnimatableProperty 来实现，每个动画内部都有此属性，用于描述 object 和动画如何进行数据沟通，即动画过程中的变化的值如何反应到object上，以及object上的值如何反应到动画的状态上。 pop 库内置了 UIKit 和 Cocoa 中大部分对象相关的动画属性，我们可以直接使用，如 kPOPLabelTextColor 就是一个对 UILabel 的文字颜色的动画属性。但是对没有内置的，我们需要自己去实现一个动画属性，这点上是保留了扩展性的，毕竟 pop 是对 NSObject 做动画的，而 NSObject 就是万物。

动画设计不是一蹴而就的，需要不断的调试， pop 内部提供了 POPAnimationTracer 用于调试跟踪某个动画。

# 动画

一切动画皆  POPAnimation, 提供了动画的操作界面，动画引擎内部都是用的这个操作界面操作动画（几个内部具体实现中可能会根据不同的动画具体类进行操作）。

然后 POPPropertyAnimation 是一个具有属性的 POPAnimation, 所以这个才具有实用价值，pop 提供给用户的四种动画(POPSpringAnimation, POPDecayAnimation, POPBasicAnimation)都是一个特殊的 POPPropertyAnimation，只是描述了某个具体曲线的动画。另外 POPCustomAnimation 是直接继承 POPAnimation 的。

每个 POPAnimation 内部有一个 _POPAnimationState 的 state, 这个属性是动画的内部状态机。每个具体的动画可能内部的状态机不同，有 _POPPropertyAnimationState, _POPBasicAnimationState, _POPSpringAnimationState, _POPDecayAnimationState 等。 事实上，所有动画相关的变化，大部分在 state 内完成。描述了开始，结束，以及每个时间帧的接口和实现。


动画是在某个 NSObject 上进行播放的。所以 pop 内部为 NSObject 添加了类别, 方便使用:


``` objective-c
@interface NSObject (POP)
// 添加动画 （添加即开始播放）
- (void)pop_addAnimation:(POPAnimation *)anim forKey:(NSString *)key;
// 移除所有动画（移除即停止播放）
- (void)pop_removeAllAnimations;
// 移除动画
- (void)pop_removeAnimationForKey:(NSString *)key;
// 获取身上的所有动画
- (NSArray *)pop_animationKeys;
// 获取某个动画
- (id)pop_animationForKey:(NSString *)key;
@end
```

内部的添加动画，移除动画等动画的管理，都是交由 POPAnimator 这个单例。这个才是 pop 的核心机器。NSObject 的类别方法，实际上是 POPAnimator 的一个面向库使用者的界面。

# 基本动画 POPBasicAnimation

基本动画给定了动画的时长和时间变化函数。

```objective-c
@interface POPBasicAnimation : POPPropertyAnimation
+ (instancetype)animationWithPropertyNamed:(NSString *)name;
@property (assign, nonatomic) CFTimeInterval duration;
@property (strong, nonatomic) CAMediaTimingFunction *timingFunction; 动画曲线
@end
```

其 state （_POPBasicAnimationState） 是继承自 _POPPropertyAnimationState 的， 所以可以指定 property 属性。

timingFunction 为描述变化bezier曲线的，内部是使用 WebCore::UnitBezier 来计算。

# 自定义动画 POPCustomAnimation

自定义动画并没有指定动画时长，是通过自定义的事件帧函数的返回值来确定动画结束的。

```objective-c
typedef BOOL (^POPCustomAnimationBlock)(id target, POPCustomAnimation *animation);
@interface POPCustomAnimation : POPAnimation
+ (instancetype)animationWithBlock:(POPCustomAnimationBlock)block;
@property (readonly, nonatomic) CFTimeInterval currentTime;
@property (readonly, nonatomic) CFTimeInterval elapsedTime;
@end
```

提供的 block 在每个时间帧都会回调，在其内部定义好每个时间点的动作即可。

# 弹簧动画 POPSpringAnimation

弹簧动画给定变化属性，自动结束动画。

```objective-c
@interface POPSpringAnimation : POPPropertyAnimation
+ (instancetype)animation;
+ (instancetype)animationWithPropertyNamed:(NSString *)name;

@property (copy, nonatomic) id velocity;
@property (assign, nonatomic) CGFloat springBounciness;
@property (assign, nonatomic) CGFloat springSpeed;
@property (assign, nonatomic) CGFloat dynamicsTension;
@property (assign, nonatomic) CGFloat dynamicsFriction;
@property (assign, nonatomic) CGFloat dynamicsMass;
@end
```

## 衰减动画 POPDecayAnimation

衰减动画给定变化属性，自动结束动画。

```objective-c
@interface POPDecayAnimation : POPPropertyAnimation
+ (instancetype)animation;
+ (instancetype)animationWithPropertyNamed:(NSString *)name;

@property (copy, nonatomic) id velocity;
@property (copy, nonatomic, readonly) id originalVelocity;
@property (assign, nonatomic) CGFloat deceleration;
@property (readonly, assign, nonatomic) CFTimeInterval duration;

- (void)setToValue:(id)toValue NS_UNAVAILABLE;
- (id)reversedVelocity;
@end

```

# WebCore
内置了 WebCore 的几种算法，都放在 WebCore 文件夹下了。包括 UnitBezier, FloatConversion, TransformationMatrix 三个算法。

# 时间帧
在一个事件侦内，使用 [CATrasaction begin] 和 [CATrasaction commit] 来包裹的。但是设置 [CATransaction setDisableActions:YES], 因为没帧内，不需要做 core animation 动画。

每一帧内对动画对象进行更新。通过调用 POPAnimation 内的 state 来具体更新。

# 其他

## 生成一个临时的唯一表示符

在添加一个动画的时候，如果key为nil，内部会为其自动生成一个唯一标识符。

[[NSUUID UUID] UUIDString] 每次调用都会返回一个新的唯一表示符。


## CADisplayLink

CADisplayLink是一个能让我们以和屏幕刷新率相同的频率将内容画到屏幕上的定时器。我们在应用中创建一个新的 CADisplayLink 对象，把它添加到一个runloop中，并给它提供一个 target 和selector 在屏幕刷新的时候调用。在 POPAnimator 内部是用 CADisplayLink 来驱动的。

iOS设备的屏幕刷新频率是固定的，CADisplayLink在正常情况下会在每次刷新结束都被调用，精确度相当高。所以在自定义动画，或视频播放的渲染用 CADisplayLink 比较合适。反观 NSTimer, 可能因为 runloop 而导致精确度低。

## OSSpinLock
在 POPAnimator 内部使用 OSSpinLock 的锁来管理内部的动画列表的。称为自旋锁，是 ios 上目前效率最高的锁，主要是因为其实现并没有进入系统内核层，可以节省系统调用和上下文切换。所以在一些开源项目中用得比较多。另外 synchronized 和 NSConditionLock 是性能最差的锁。可以参看 http://www.2cto.com/kf/201505/400344.html

```c
OSSpinLock _lock = OS_SPINLOCK_INIT;
OSSpinLockLock(&_lock);
// 锁区操作
OSSpinLockUnlock(&_lock);
```


## __VA_ARGS__ 用于宏之间传递可变参数

```c
#define FB_PROPERTY_SET(stype, property, mutator, ctype, ...) \
- (void)mutator (ctype)value { \
  if (value == ((stype *)_state)->property) \
    return; \
  ((stype *)_state)->property = value; \
  __VA_ARGS__ \
}

#define DEFINE_RW_PROPERTY(stype, flag, mutator, ctype, ...) \
  FB_PROPERTY_GET (stype, flag, ctype) \
  FB_PROPERTY_SET (stype, flag, mutator, ctype, __VA_ARGS__)

// 用法
DEFINE_RW_PROPERTY_OBJ(POPPropertyAnimationState, property, setProperty:, POPAnimatableProperty*, ((POPPropertyAnimationState
```


## 贝塞尔曲线

根据四个位置任意的点坐标绘制出的一条光滑曲线。由法国数学家Pierre Bézier发明，并给出了计算公式。可以扩展到多个控制点。

特别的，当只提供两个点：起点和终点的时候，是一个直线。


## CAMediaTimingFunction

描述时间曲线，函数将在区间 [0, 1] 的时间映射到另一个 [0,1] 区间内的一个时间。

时间的 bezier 曲线是从 (0,0) 到 (1,1) 点，中间有两个控制点。控制点是自定义的。

内置的几种时间函数:

- Linear  控制点： (0,0), (1,1)
- EaseIn 控制点:  (0.42,0), (1,1)
- EaseOut 控制点：(0,0), (0.58,1.0)
- EaseInEaseOut 控制点：(0.42,0), (0.58,1)
- Default 控制点：(0.25,0.1) and (0.25,1)

```objective-c
// 内置的时间函数: kCAMediaTimingFunctionLinear 等
+ (instancetype)functionWithName:(NSString *)name
// 这种语法是合法的。 贝塞尔曲线自动调整为从 (0, 0) 到 (1, 1), 之间的两个控制点为 (c1x, c1y), (c2x, c2y)
- (id)initWithControlPoints:(float)c1x
                           :(float)c1y
                           :(float)c2x
                           :(float)c2y;
						   
// 获取控制点, index取值0~3
- (void)getControlPointAtIndex:(size_t)index
                        values:(float [2])ptr							   
```

