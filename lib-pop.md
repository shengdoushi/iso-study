# pop 动画库

# 概述

项目地址: https://github.com/facebook/pop/tree/master/pop

是一个动画引擎，提供了基本的动画和诸如 spring, decay 等高级动画。是著名应用 Paper 的动画引擎。因为 pop 是一个 ios, osx, tvos 三个平台上的库，下面只针对ios平台。

当前的版本为 1.0.9,  commit为 b30efd8。


# 用法
1. 构造一个动画，共有四种类型 spring, decay, basic, 和 custom。 分别对应实现类: POPSpringAnimation, POPDecayAnimation, POPBasicAnimation 和 POPCustomAnimation. 每种动画有自己的变量，无非就是设置动画的初始状态，和动画的结束状态，以及描述动画过程中变化曲线的属性。

```objective-c
POPSpringAnimation *anim = [POPSpringAnimation animation];
```

2. 启动动画，只要将动画添加到要执行的对象(NSObject)上: 通过 [object pop_addAnimation:anim forKey:@"key"]，之后可以通过 [object pop_animationForkey:@"key"] 来获取动画，更新内部的动画属性

```objective-c
[layer pop_addAnimation:anim forKey:@"myKey"];

anim = [layer pop_animationForKey:@"myKey"];
if (anim) {
  /* update to value to new destination */
  anim.toValue = @(42.0);
} else {
  /* create and start a new animation */
  ....
}
```

3. 停止动画，将动画从对象身上移除： [object pop_removeAnimationForKey:@"key"]


```objective-c
[layer pop_removeAnimationForKey:@"myKey"];
```

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

# 属性动画 POPPropertyAnimation

```objective-c
@interface POPPropertyAnimation : POPAnimation
@property (strong, nonatomic) POPAnimatableProperty *property; // 属性

/**
动画初始值， 如果未指定，则默认为 object 的当前属性值
 */
@property (copy, nonatomic) id fromValue;

/**
动画结束值， 如果未指定，则默认为 object 的当前属性值
 */
@property (copy, nonatomic) id toValue;

/**
动画的当前值的舍入因子。1.0 表示在整数值间舍入，0表示没有舍入。
 */
@property (assign, nonatomic) CGFloat roundingFactor;

/**
动画的当前值的约束模式。默认不约束(kPOPAnimationClampNone), 共有四种约束： 不约束，根据开始值约束，根据结束值约束，约束到开始值和结束值之间。
*/
@property (assign, nonatomic) NSUInteger clampMode;
/**
动画的值是否是添加到每一帧，还是设置到某一帧。默认是NO，即设置模式
*/
@property (assign, nonatomic, getter = isAdditive) BOOL additive;

@end
```

property 动画，主要分为开始值 fromValue，结束值 toValue，以及对当前值的约束规则。以及 property 描述了动画的值和object 的值得对应关系。

roundingFactor 是一个数值舍入因子。 可以认为是舍入到小数点后几位，如 10 表示舍入到小数点后1位。舍入流程是先对值和因子的乘积四舍五入，然后除以因子， 代码如下:

```objective-c
NS_INLINE CGFloat POPSubRound(CGFloat f, CGFloat sub/*舍入因子*/)
{
  return round(f * sub) / sub;
}
```
	
# 基本动画 POPBasicAnimation

基本动画给定了动画的时长和时间变化函数。

```objective-c
@interface POPBasicAnimation : POPPropertyAnimation
+ (instancetype)animationWithPropertyNamed:(NSString *)name;
@property (assign, nonatomic) CFTimeInterval duration; // 动画时长
@property (strong, nonatomic) CAMediaTimingFunction *timingFunction; // 动画曲线
@end
```

其 state （_POPBasicAnimationState） 是继承自 _POPPropertyAnimationState 的， 所以可以指定 property 属性。

基本动画需要设置两个属性 duration 和 timingFunction。
timingFunction 为描述变化bezier曲线的，内部是使用 WebCore::UnitBezier 来计算。

# 自定义动画 POPCustomAnimation

自定义动画并没有指定动画时长，是通过自定义的事件帧函数的返回值来确定动画结束的。

```objective-c
typedef BOOL (^POPCustomAnimationBlock)(id target, POPCustomAnimation *animation);
@interface POPCustomAnimation : POPAnimation
+ (instancetype)animationWithBlock:(POPCustomAnimationBlock)block;
@property (readonly, nonatomic) CFTimeInterval currentTime; // 当前回调的时间
@property (readonly, nonatomic) CFTimeInterval elapsedTime; // 从上次回调到当前回调的时间间隔
@end
```

自定义动画只需要设置一个 block 回调参数。提供的 block 在每个时间帧都会回调，在其内部定义好每个时间点的动作即可。

# 弹簧动画 POPSpringAnimation

弹簧动画给定变化属性，自动结束动画。

```objective-c
@interface POPSpringAnimation : POPPropertyAnimation
+ (instancetype)animation;
+ (instancetype)animationWithPropertyNamed:(NSString *)name;

@property (copy, nonatomic) id velocity; // 当前速度，在动画开始前设置
/**
有效的反弹振幅
[0,20] 默认: 4， 值越大，会导致弹簧振幅的范围变大，弹性效果越明显。
与 springSpeed 结合使用，
*/
@property (assign, nonatomic) CGFloat springBounciness;
/**
有效的速度
[0,20], 默认: 12, 值越大，对弹簧的抑制效果越明显，导致初始速度变大，动画结束的也越快
与 springBounciness
*/
@property (assign, nonatomic) CGFloat springSpeed; // 速度

// 下边三个主要在动力学模拟中用到
@property (assign, nonatomic) CGFloat dynamicsTension; // 张力
@property (assign, nonatomic) CGFloat dynamicsFriction; // 摩擦
@property (assign, nonatomic) CGFloat dynamicsMass; // 质量
@end
```

velocity 是一个计算型变量，返回的是 POPBox(__state->velocityVec, __state->valueType), 设置的时候直接影响 __state 内的 velocityVec 和 originalVelocityVec 变量，所以设置的话，需要在动画开始前设置，否则会影响动画内部状态。

spring 动画一般主要设置三个属性：velocity, springBounciness 和 springSpeed。

关于动力学模拟，这里主要是最后三个属性 dynamicsTension, dynamicsFriction, 和 dynamicsMass, 一旦设置了任一一个，都会开启动力学模拟。这些参数和 springSpeed, springBounciness 属性是互斥的。当设置了 springBounciness， springSpeed 任一一个，都会关闭动力学模拟。

尽管上边两组参数是互斥的， 但是当关闭动力学模拟时候，内部实现中会转换为动力学模拟的三个参数: 张力，摩擦和质量。因为最底层的 POP::SpringSolver 内部spring计算中还是使用的动力学模拟的三个参数作为参数，来计算spring动画的。

另外， UIKit 内 UIView 上有个 spring 动画的方法（iOS 7+):

```objective-c
+ (void)animateWithDuration:(NSTimeInterval)duration
                      delay:(NSTimeInterval)delay
     usingSpringWithDamping:(CGFloat)dampingRatio
      initialSpringVelocity:(CGFloat)velocity
                    options:(UIViewAnimationOptions)options
                 animations:(void (^)(void))animations
                 completion:(void (^)(BOOL finished))completion
```

只提供了 dampingRatio 和 velocity 两个属性可调。 dampingRatio 影响动画在最后的振幅，成为阻尼系数，[0,1], 值越大，阻力越大，则振幅越小。 velocity 代表初始速度，[0,1] 是一个速度的百分比，比如动画的总移动距离是200 pt， 你希望速度是 100pt/s, 那么 velocity 就设置为 0.5，另一方面， velocity 因为是一个百分比，所以间接的就影响了动画的时长，如 0.5 下，动画时长就是 2s。

## 衰减动画 POPDecayAnimation

衰减动画给定变化属性，自动结束动画。

```objective-c
@interface POPDecayAnimation : POPPropertyAnimation
+ (instancetype)animation;
+ (instancetype)animationWithPropertyNamed:(NSString *)name;

/**
速度，动画开始前设置。同 POPSpringAnimation 中的 velocity
*/
@property (copy, nonatomic) id velocity;
/**
readonly 变量
本次动画开始速度，因为动画可能 翻转，或重播。
*/
@property (copy, nonatomic, readonly) id originalVelocity;
/**
[0,1] 默认： 0.998 值越小，表示更快的减速
*/
@property (assign, nonatomic) CGFloat deceleration;
/**
readonly 变量
动画的期望时长
*/
@property (readonly, assign, nonatomic) CFTimeInterval duration;

- (id)reversedVelocity;
@end

```

可以看到 decay 动画主要设置两个属性: velocity, deceleration。

# WebCore
内置了 WebCore 的几种算法，都放在 WebCore 文件夹下了。包括 UnitBezier, FloatConversion, TransformationMatrix 三个算法。

# 时间帧
在一个事件侦内，使用 [CATrasaction begin] 和 [CATrasaction commit] 来包裹的。但是设置 [CATransaction setDisableActions:YES], 因为没帧内，不需要做 core animation 动画。

每一帧内对动画对象进行更新。通过调用 POPAnimation 内的 state 来具体更新。

```objective-c
- (void)_renderTime:(CFTimeInterval)time item:(POPAnimatorItemRef)item
{
  id obj = item->object;
  POPAnimation *anim = item->animation;
  POPAnimationState *state = POPAnimationGetState(anim);

  if (nil == obj) {
    // 没有 object ， 则清理掉并返回
    NSAssert(item->unretainedObject, @"object should exist");
    stopAndCleanup(self, item, true, false);
  } else {

    // 启动动画（如果需要的话）
    state->startIfNeeded(obj, time, _slowMotionAccumulator);

    // 暂停的动画不播放
    if (state->active && !state->paused) {
      // object exists; animate
      applyAnimationTime(obj, state, time);

      FBLogAnimDebug(@"time:%f running:%@", time, item->animation);
      if (state->isDone()) {
        // set end value
        applyAnimationToValue(obj, state);

        state->repeatCount--;
        if (state->repeatForever || state->repeatCount > 0) {
          if ([anim isKindOfClass:[POPPropertyAnimation class]]) {
            POPPropertyAnimation *propAnim = (POPPropertyAnimation *)anim;
            id oldFromValue = propAnim.fromValue;
            propAnim.fromValue = propAnim.toValue;

            if (state->autoreverses) {
              if (state->tracing) {
                [state->tracer autoreversed];
              }

              if (state->type == kPOPAnimationDecay) {
                POPDecayAnimation *decayAnimation = (POPDecayAnimation *)propAnim;
                decayAnimation.velocity = [decayAnimation reversedVelocity];
              } else {
                propAnim.toValue = oldFromValue;
              }
            } else {
              if (state->type == kPOPAnimationDecay) {
                POPDecayAnimation *decayAnimation = (POPDecayAnimation *)propAnim;
                id originalVelocity = decayAnimation.originalVelocity;
                decayAnimation.velocity = originalVelocity;
              } else {
                propAnim.fromValue = oldFromValue;
              }
            }
          }

          state->stop(NO, NO);
          state->reset(true);

          state->startIfNeeded(obj, time, _slowMotionAccumulator);
        } else {
          stopAndCleanup(self, item, state->removedOnCompletion, YES);
        }
      }
    }
  }
}

```

# 其他

## NSUUID 生成一个临时的唯一表示符

在添加一个动画的时候，如果key为nil，内部会为其自动生成一个唯一标识符。

[[NSUUID UUID] UUIDString] 每次调用都会返回一个新的唯一表示符。


## CADisplayLink

CADisplayLink是一个能让我们以和屏幕刷新率相同的频率将内容画到屏幕上的定时器。我们在应用中创建一个新的 CADisplayLink 对象，把它添加到一个runloop中，并给它提供一个 target 和selector 在屏幕刷新的时候调用。在 POPAnimator 内部是用 CADisplayLink 来驱动的。

iOS设备的屏幕刷新频率是固定的，CADisplayLink在正常情况下会在每次刷新结束都被调用，精确度相当高。所以在自定义动画，或视频播放的渲染用 CADisplayLink 比较合适。反观 NSTimer, 可能因为 runloop 而导致精确度低。

## OSSpinLock 自旋锁
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

## CACurrentMediaTime() 获取当前时间

函数原型：

```c
CFTimeInterval CACurrentMediaTime ( void );
```

内部调用 mach_absolute_time()  获取当前时间转换为秒并返回。

mach_absolute_time 是一个CPU/总线依赖函数，返回一个基于系统启动后的时钟”嘀嗒”数(int64),属于高精度的纳秒级别。
模拟下函数实现:

```c
CFTimeInterval CACurrentMediaTime(){
	int64_t time = mach_absolute_time();
    mach_timebase_info_data_t timebase;
    mach_timebase_info(&timebase);
    return (CFTimeInterval)time * (double)timebase.numer / (double)timebase.denom /1e9;
}
```


## NS_UNAVAILABLE  标记不可用

标记一个方法不可用, 方法声明中放在尾部：

```objective-c
- (void)setToValue:(id)toValue NS_UNAVAILABLE;
```

可以在某些不允许调用某些方法的场景下使用，例如我们提供了一个带参数的init方法，但是又不允许使用者使用默认的init方法, 可以使用 NS_UNAVAILABLE 限定下, 将管理工作交给编译器:

```
@interface MyObject : NSObject
-(instancetype)init NS_UNAVAILABLE;
@end
```

类似的还有 NS_AUTOMATED_REFCOUNT_UNAVAILABLE 表示在 arc 选项下这些函数不可用:

```
- (id)retain NS_AUTOMATED_REFCOUNT_UNAVAILABLE;
- (oneway void)release NS_AUTOMATED_REFCOUNT_UNAVAILABLE;
- (id)autorelease NS_AUTOMATED_REFCOUNT_UNAVAILABLE;
- (NSUInteger)retainCount NS_AUTOMATED_REFCOUNT_UNAVAILABLE;
```
