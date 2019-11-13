---
title: iOS-转场动画
date: 2017-05-23 18:49:44
categories: "iOS"
tags: iOS
---


### 学习及实践笔记
记录iOS中常见Modal、Push转场动画的学习及实践
<!--more-->


#### 相关api的记录及介绍

<strong> [喵神文章传送门](https://onevcat.com/2013/10/vc-transition-in-ios7/) </strong>
```
/*这个接口用来提供切换上下文给开发者使用，包含了从哪个VC到哪个VC等各类信息*/
@protocol UIViewControllerContextTransitioning

-(UIView *)containerView; VC切换所发生的view容器，开发者应该将切出的view移除，将切入的view加入到该view容器中。
-(UIViewController *)viewControllerForKey:(NSString *)key; 提供一个key，返回对应的VC。现在的SDK中key的选择只有UITransitionContextFromViewControllerKey和UITransitionContextToViewControllerKey两种，分别表示将要切出和切入的VC。
-(CGRect)initialFrameForViewController:(UIViewController *)vc; 某个VC的初始位置，可以用来做动画的计算。
-(CGRect)finalFrameForViewController:(UIViewController *)vc; 与上面的方法对应，得到切换结束时某个VC应在的frame。
-(void)completeTransition:(BOOL)didComplete; 向这个context报告切换已经完成。


/* 自定义转场动画中使用 */
/*这个接口负责切换的具体内容，也即“切换中应该发生什么”。开发者在做自定义切换效果时大部分代码会是用来实现这个接口*/
@protocol UIViewControllerAnimatedTransitioning

-(NSTimeInterval)transitionDuration:(id < UIViewControllerContextTransitioning >)transitionContext; 系统给出一个切换上下文，我们根据上下文环境返回这个切换所需要的花费时间

-(void)animateTransition:(id < UIViewControllerContextTransitioning >)transitionContext; 在进行切换的时候将调用该方法，我们对于切换时的UIView的设置和动画都在这个方法中完成。


/* ViewController中调用 */
@protocol UIViewControllerTransitioningDelegate

// 动画
-(id< UIViewControllerAnimatedTransitioning >)animationControllerForPresentedController:(UIViewController *)presented presentingController:(UIViewController *)presenting sourceController:(UIViewController *)source;
-(id< UIViewControllerAnimatedTransitioning >)animationControllerForDismissedController:(UIViewController *)dismissed;

// 交互
-(id< UIViewControllerInteractiveTransitioning >)interactionControllerForPresentation:(id < UIViewControllerAnimatedTransitioning >)animator;
-(id< UIViewControllerInteractiveTransitioning >)interactionControllerForDismissal:(id < UIViewControllerAnimatedTransitioning >)animator;


/* UIViewControllerInteractiveTransitioning 提供百分比控制交互切换*/

-(Float)completionSpeed 返回速度  1 - self.percentComplete;

-(void)updateInteractiveTransition:(CGFloat)percentComplete 更新百分比，一般通过手势识别的长度之类的来计算一个值，然后进行更新。之后的例子里会看到详细的用法
-(void)cancelInteractiveTransition 报告交互取消，返回切换前的状态
–(void)finishInteractiveTransition 报告交互完成，更新到切换后的状态

```
* 注意点
  locationInView:获取到的是手指点击屏幕实时的坐标点；
  translationInView：获取到的是手指移动后，在相对坐标中的偏移量

---
#### Modal

<strong>注意：当我们在被modal出的控制器中，向self发送dismissViewController的方法时，这个消息，会被直接转发到显示它的VC中去</strong>



##### Present
<strong>BouncePresentTransition的具体实现</strong>
```
#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>

@interface SPBouncePresentTransition : NSObject<UIViewControllerAnimatedTransitioning>

@end

```
```
#import "SPBouncePresentTransition.h"

@implementation SPBouncePresentTransition

- (NSTimeInterval)transitionDuration:(id<UIViewControllerContextTransitioning>)transitionContext{

    return 0.8f;
    
}

- (void)animateTransition:(id<UIViewControllerContextTransitioning>)transitionContext{

    // 获取相关ViewController
    UIViewController *toVC = [transitionContext viewControllerForKey:UITransitionContextToViewControllerKey];
    // UIViewController *fromVC = [transitionContext viewControllerForKey:UITransitionContextFromViewControllerKey];
    
    CGRect bounds = [UIScreen mainScreen].bounds;
    // 设置被modal出界面view的frame
    CGRect finalFrame = [transitionContext finalFrameForViewController:toVC];
    // 设置目标界面frame的offset，即初始位置
    toVC.view.frame = CGRectOffset(finalFrame, 0, bounds.size.height);
    
    // 取出containerView，将目标vc的view进行添加
    UIView *containerView = [transitionContext containerView];
    [containerView addSubview:toVC.view];
    
    // 设置view的frame动画效果
    [UIView animateWithDuration:[self transitionDuration:transitionContext] delay:0.0 usingSpringWithDamping:0.6 initialSpringVelocity:0.0 options:UIViewAnimationOptionCurveLinear animations:^{
        toVC.view.frame = finalFrame;
    } completion:^(BOOL finished) {
        // 提交transion完成
        [transitionContext completeTransition:YES];
    }];
}

@end
```
**在viewcontroller中如何使用**

1.遵守`UIViewControllerTransitioningDelegate`协议
2.设置modal的vc的`transitioningDelegate`属性
3.实现代理方法
```
-(id<UIViewControllerAnimatedTransitioning>)animationControllerForPresentedController:(UIViewController *)presented presentingController:(UIViewController *)presenting sourceController:(UIViewController *)source
```

具体可参照如下代码
```
#pragma mark - Transition
- (void)testForModalTransition{

    SPModalViewController *vc = [[SPModalViewController alloc] init];
    vc.transitioningDelegate = self;// 设置代理
    vc.dismissAction = ^{
        [self dismissViewControllerAnimated:YES completion:nil];
    };
    [self presentViewController:vc animated:YES completion:nil];
    
}

// 代理方法
// init
/* self.bouncePresent = [[SPBouncePresentTransition alloc]init]; */
-(id<UIViewControllerAnimatedTransitioning>)animationControllerForPresentedController:(UIViewController *)presented presentingController:(UIViewController *)presenting sourceController:(UIViewController *)source{

    return self.bouncePresent;
    
}
```
效果如下

![animateTransition.gif](http://upload-images.jianshu.io/upload_images/1742463-686df56e8009359f.gif?imageMogr2/auto-orient/strip)

##### Dismiss
<strong>滑动效果的转场实现</strong>
需要注意的是，务必同时实现两个代理方法
触发顺序，为`动画类->交互类`,系统会优先处理`animationControllerForDismissedController`方法，如果有，则dismiss操作均为自定义dismissTransition（demo中的transitionDuration已经设置为1.5s,效果可如gif动图所示），如果没有设置，那么则为系统的dismiss动画。
```
// 动画类
- (id<UIViewControllerAnimatedTransitioning>)animationControllerForDismissedController:(UIViewController *)dismissed
```
```
// 交互类
- (id<UIViewControllerInteractiveTransitioning>)interactionControllerForDismissal:(id<UIViewControllerAnimatedTransitioning>)animator
```
具体可参照如下代码

1.NormalDismiss
```
#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>

@interface SPNormalDismissTransition : NSObject<UIViewControllerAnimatedTransitioning>

@end
```
```
#import "SPNormalDismissTransition.h"

@implementation SPNormalDismissTransition

- (NSTimeInterval)transitionDuration:(id<UIViewControllerContextTransitioning>)transitionContext{

    return 1.5f;
    
}

- (void)animateTransition:(id<UIViewControllerContextTransitioning>)transitionContext{

    UIViewController *fromVC = [transitionContext viewControllerForKey:UITransitionContextFromViewControllerKey];
    UIViewController *toVC = [transitionContext viewControllerForKey:UITransitionContextToViewControllerKey];
    
    CGRect initialFrame = [transitionContext initialFrameForViewController:fromVC];
    CGRect finalFrame = CGRectOffset(initialFrame, 0, [UIScreen mainScreen].bounds.size.height);
    
    UIView *containerView  = [transitionContext containerView];// 承载当前活动vc的view的containerView
    [containerView addSubview:toVC.view];
    [containerView sendSubviewToBack:toVC.view];// 将视图置于下层
    
    
    [UIView animateWithDuration:[self transitionDuration:transitionContext] animations:^{
        fromVC.view.frame = finalFrame;
    } completion:^(BOOL finished) {
        [transitionContext completeTransition:![transitionContext transitionWasCancelled]];
    }];
    
}

@end
```
2.SwipeTransition
```
#import <UIKit/UIKit.h>

@interface SPPercentSwipeTransition : UIPercentDrivenInteractiveTransition

@property (nonatomic, assign) BOOL interacting;// 用来判断是否处理交互
- (void)addSwipeTransition2viewController:(UIViewController *)viewController;// 传入要添加panRecognizer的viewController

@end
```
```
#import "SPPercentSwipeTransition.h"

@interface SPPercentSwipeTransition()

@property (nonatomic, weak) UIViewController *presenedController;// 当前被modal出的控制器 注意weak的使用，强引用造成无法释放
@property (nonatomic, assign) BOOL canEndTransition;// 当超过屏幕高度的1/4时，我们认为交互可以结束

@end

@implementation SPPercentSwipeTransition

- (void)addSwipeTransition2viewController:(UIViewController *)viewController{

    self.presenedController = viewController;
    [self addRecognizer2View:viewController.view];
    
}

- (void)addRecognizer2View:(UIView *)view{

    UIPanGestureRecognizer *pan = [[UIPanGestureRecognizer alloc] init];
    [view addGestureRecognizer:pan];
    [pan addTarget:self action:@selector(handleRecognizer:)];
    
}

- (void)handleRecognizer:(UIPanGestureRecognizer *)recognizer{

    CGRect screenBounds = [UIScreen mainScreen].bounds;
    CGPoint point = [recognizer translationInView:recognizer.view.superview];// translationInView
    switch (recognizer.state) {
        case UIGestureRecognizerStateBegan:
            self.interacting = YES;
            [self.presentingController dismissViewControllerAnimated:YES completion:nil];
            break;
        case UIGestureRecognizerStateChanged:
        {
            CGFloat actionPoint = point.y/(screenBounds.size.height /2);// 有效距离为屏幕高度的一半
            actionPoint = fminf(fmaxf(actionPoint, 0.0), 1.0);
            self.canEndTransition = actionPoint > 0.5;// 滑动超过屏幕高度的1/4则可完成转场
            [self updateInteractiveTransition:actionPoint];
        }
            break;
        case UIGestureRecognizerStateEnded:
        case UIGestureRecognizerStateCancelled:
        {
            self.interacting = NO;
            if (self.canEndTransition)
            {
                [self finishInteractiveTransition];
            }
            else if (!self.canEndTransition || recognizer.state == UIGestureRecognizerStateCancelled)
            {
                [self cancelInteractiveTransition];
            }
        }
            break;
        default:
            break;
    }
    
}

- (CGFloat)completionSpeed{// 返回转场的速度

    return 1 - self.percentComplete;
    
}
```
效果如下

![swipeTransition.gif](http://upload-images.jianshu.io/upload_images/1742463-d2084692ba438658.gif?imageMogr2/auto-orient/strip)

##### SemiModal

实现一个常见的semi半挂式modal，将我们上述的AnimationTransition(present)、NormalDismiss部分稍作处理即可
效果如下：

![semiModal.gif](http://upload-images.jianshu.io/upload_images/1742463-7e61ef131640c488.gif?imageMogr2/auto-orient/strip)

首先分析实现思路：
1.semi部分，在`bouncePresentTransition`中的toVC的view进行圆角处理，同时设置它在containerView中的frame为目标样式
2.背景部分，将当前fromVC的view进行截图，添加到转场容器`containerView`中，然后改变它的`transform`中的scale属性，进行等比例缩放，同时将底层viewController的view进行隐藏处理
3.因为上文中，我们已经处理了基于`UIPercentDrivenInteractiveTransition`的`swipeTransition`，所以不需要再关心百分比变化
4.上文中处理的`normalDismiss`，只是简单的对视图进行了切换，因为考虑到semi的样式会对底层viewController的view进行隐藏处理，所以我们需要特别注意，在transition完成时，要将目标vc的view显示出来

思路有了，那么接下来就是实现，具体请看代码：

<strong>截图方法 </strong>
```
- (UIImage *)getSnapShotFromView:(UIView *)view{
    
    CGRect rect = view.bounds;
    UIGraphicsBeginImageContextWithOptions(rect.size, false, 0);
    [view.layer renderInContext:UIGraphicsGetCurrentContext()];
    UIImage *snapShot = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return snapShot;
    
}
```

<strong>改造的bouncePresent</strong>
```
- (void)animateTransition:(id<UIViewControllerContextTransitioning>)transitionContext{

    // 获取相关ViewController
    UIViewController *toVC = [transitionContext viewControllerForKey:UITransitionContextToViewControllerKey];
    UIViewController *fromVC = [transitionContext viewControllerForKey:UITransitionContextFromViewControllerKey];
    
    CGRect bounds = [UIScreen mainScreen].bounds;
    // 设置被modal出界面view的frame
//    CGRect finalFrame = [transitionContext finalFrameForViewController:toVC];
    
    // -- semi效果 --
    CGRect finalFrame = CGRectMake(0, bounds.size.height/3, bounds.size.width, bounds.size.height);
    
    // 设置目标界面frame的offset，即初始位置
    toVC.view.frame = CGRectOffset(finalFrame, 0, bounds.size.height);
    
    // -- semi效果 --
    toVC.view.layer.cornerRadius = 15;
    [toVC.view.layer masksToBounds];
    
    UIImageView *snapView = [[UIImageView alloc] initWithImage:[self getSnapShotFromView:fromVC.view]];
    snapView.frame = fromVC.view.bounds;
    snapView.tag = 404;//标记一下
    
    // 将fromvc的view隐藏掉
    fromVC.view.hidden = YES;
    
    // 取出containerView(视图管理容器)，将目标vc的view进行添加
    UIView *containerView = [transitionContext containerView];
    // -- semi效果 --
    [containerView addSubview:snapView];
    // 注意添加顺序
    [containerView addSubview:toVC.view];
    
    // 设置view的frame动画效果
    [UIView animateWithDuration:[self transitionDuration:transitionContext] delay:0.0 usingSpringWithDamping:0.6 initialSpringVelocity:0.0 options:UIViewAnimationOptionCurveLinear animations:^{
        
        toVC.view.frame = finalFrame;
        snapView.transform = CGAffineTransformMakeScale(0.85, 0.85);
    
    } completion:^(BOOL finished) {
        
        // 提交transion完成
        if ([transitionContext transitionWasCancelled]) {
            [transitionContext completeTransition:NO];
            [snapView removeFromSuperview];
            fromVC.view.hidden = NO; // 注意隐藏操作
        }else{
            [transitionContext completeTransition:YES];
        }
        
    }];
}
```
<strong>改造的normalDismiss</strong>
```
- (void)animateTransition:(id<UIViewControllerContextTransitioning>)transitionContext{

    UIViewController *fromVC = [transitionContext viewControllerForKey:UITransitionContextFromViewControllerKey];
    UIViewController *toVC = [transitionContext viewControllerForKey:UITransitionContextToViewControllerKey];
    
    CGRect initialFrame = [transitionContext initialFrameForViewController:fromVC];
    CGRect finalFrame = CGRectOffset(initialFrame, 0, [UIScreen mainScreen].bounds.size.height);
    
    
    UIView *containerView  = [transitionContext containerView];// 承载当transitionview的containerView
    UIView *snapView = nil;// 取出截图 tag = 404
    for (UIView *sub in containerView.subviews) {
        if (sub.tag == 404) {
            snapView = sub;
            break;
        }
    }
    [containerView addSubview:toVC.view];
    [containerView sendSubviewToBack:toVC.view];// 将视图置于下层
    
    
    
    
    [UIView animateWithDuration:[self transitionDuration:transitionContext] animations:^{
        
        fromVC.view.frame = finalFrame;
        snapView.transform = CGAffineTransformIdentity;
        
    } completion:^(BOOL finished) {
        
        BOOL transitionComplete = ![transitionContext transitionWasCancelled];
        [transitionContext completeTransition:transitionComplete];
        
        if (transitionComplete) {
            toVC.view.hidden = NO;
            [snapView removeFromSuperview];
        }else{
            toVC.view.hidden = YES;
        }
        
        
    }];
    
}
```
---
#### Push&Pop

##### Push
<strong>在viewcontroller中如何使用</strong>

1.遵守`UINavigationControllerDelegate`协议
2.设置`viewcontroller.navigationController.delegate`的代理
3.实现代理方法
```
- (id<UIViewControllerAnimatedTransitioning>)navigationController:(UINavigationController *)navigationController animationControllerForOperation:(UINavigationControllerOperation)operation fromViewController:(UIViewController *)fromVC toViewController:(UIViewController *)toVC
```
这里需要**特别注意**一点，对于Push和Pop，都会在此方法中返回，需要我们对`operation`进行类型判断，返回正确类型的自定义transition
同时要考虑一种特殊情况，就是自己并非第一层viewController时，从自身pop出去的情况
```
if (fromVC == self && operation == UINavigationControllerOperationPop)
 {// 从自己pop出去
     return nil;
 }
```

（我们可以发现，Push同上文中提到的Modal的处理方式一致，只是遵守的协议不同）

先看效果

![transPush.gif](http://upload-images.jianshu.io/upload_images/1742463-69256237d021e827.gif?imageMogr2/auto-orient/strip)

分析一下思路：
1.将点击的Cell中的imageView，传入到转场容器container中，转换其坐标、截图并添加（注意同toVC.view的添加顺序），我们命名为snapView
2.获取fromVC中的目标图片位置，设置为snapview的终点frame

大概看一下我们如何实现，写一下几个比较关键的地方

```
// 1.fromVC，处理点击CollectionView的点击事件
- (void)collectionView:(UICollectionView *)collectionView didSelectItemAtIndexPath:(NSIndexPath *)indexPath{

    PushCell *cell = (PushCell *)[collectionView cellForItemAtIndexPath:indexPath];
    self.transImgView = cell.transImg;
    
    SPGakkiViewController *gakki = [[SPGakkiViewController alloc] init];
    [self.navigationController pushViewController:gakki animated:YES];
    
}

// 2.transition中的写法
SPGakkiViewController *toVC = [transitionContext viewControllerForKey:UITransitionContextToViewControllerKey];
    SPPushViewController *fromVC = [transitionContext viewControllerForKey:UITransitionContextFromViewControllerKey];
    
    // 获取视图容器
    UIView *containerView = [transitionContext containerView];
    
    // 获取需要变换的视图
    UIImageView *transImgView = fromVC.transImgView;
    UIView *snapView = [transImgView snapshotViewAfterScreenUpdates:NO];
    snapView.frame = [transImgView convertRect:transImgView.bounds toView:containerView];
    
    // 先隐藏视图
    toVC.showContent = NO;

    [containerView addSubview:toVC.view];
    [containerView addSubview:snapView];
    
    // 动画部分
    [UIView animateWithDuration:[self transitionDuration:transitionContext] delay:0 usingSpringWithDamping:0.6 initialSpringVelocity:1/0.6 options:UIViewAnimationOptionCurveEaseInOut animations:^{
        
        snapView.frame = toVC.targetFrame;
        
    } completion:^(BOOL finished) {
        
        snapView.hidden = YES;
        toVC.showContent = YES;
        [snapView removeFromSuperview];
        [transitionContext completeTransition:![transitionContext transitionWasCancelled]];
        
    }];

```

##### Pop
效果如下

![transPop.gif](http://upload-images.jianshu.io/upload_images/1742463-0223c8489aa33444.gif?imageMogr2/auto-orient/strip)

如果一路写到这里，想必已经明白转场动画的基本使用，所以我们不再赘述，只附效果的实现代码
```
SPPushViewController *toVC = [transitionContext viewControllerForKey:UITransitionContextToViewControllerKey];
     SPGakkiViewController *fromVC = [transitionContext viewControllerForKey:UITransitionContextFromViewControllerKey];
    
    // 获取视图容器
    UIView *containerView = [transitionContext containerView];
    [containerView addSubview:toVC.view];
    toVC.view.alpha = 0;
   
    // 截图
    UIView *snapView = [fromVC.imageView snapshotViewAfterScreenUpdates:NO];

    // 隐藏
    fromVC.showContent = NO;
    
    // 目标位置
    CGRect startFrame = fromVC.targetFrame;
    CGRect targetFrame = [toVC.transImgView convertRect:toVC.transImgView.bounds toView:containerView];
    snapView.frame = startFrame;
    [containerView addSubview:snapView];
    
    // 开始动画
    [UIView animateWithDuration:[self transitionDuration:transitionContext] delay:0 usingSpringWithDamping:0.6 initialSpringVelocity:1/0.6 options:UIViewAnimationOptionCurveEaseInOut animations:^{
        
        snapView.frame = targetFrame;
        snapView.alpha = 0;
        toVC.view.alpha = 1;
        
    } completion:^(BOOL finished) {
        
        snapView.hidden = YES;
        [snapView removeFromSuperview];
        [transitionContext completeTransition:![transitionContext transitionWasCancelled]];
        
    }];

```
** 注意!注意!注意! `[transitionContext completeTransition:![transitionContext transitionWasCancelled]];`务必在动画结束时上报转场结束状态（此处手动鲜血红） **

** 返回手势 **
只需要我们在之前写的`percentSwipeTransition`中稍加改造即可，我们将它稍加封装，代码如下

```
#import <UIKit/UIKit.h>

typedef NS_ENUM(NSInteger,SPTransitionType) {

    SPTransitionTypePush,
    SPTransitionTypeModal
    
};

@interface SPPercentSwipeTransition : UIPercentDrivenInteractiveTransition

@property (nonatomic, assign) BOOL interacting;

- (void)addSwipeTransition2viewController:(UIViewController *)viewController withType:(SPTransitionType)type;

@end
```
```
#import "SPPercentSwipeTransition.h"

@interface SPPercentSwipeTransition()

@property (nonatomic, weak) UIViewController *presentingController;
@property (nonatomic, assign) BOOL canEndTransition;
@property (nonatomic, assign) SPTransitionType currentType;

@end

@implementation SPPercentSwipeTransition

- (void)addSwipeTransition2viewController:(UIViewController *)viewController withType:(SPTransitionType)type{

    self.currentType = type;
    [self addSwipeTransition2viewController:viewController];
    
}

- (void)addSwipeTransition2viewController:(UIViewController *)viewController{

    self.presentingController = viewController;
    [self addRecognizer2View:viewController.view];
    
}

- (void)addRecognizer2View:(UIView *)view{

    UIPanGestureRecognizer *pan = [[UIPanGestureRecognizer alloc] init];
    [view addGestureRecognizer:pan];
    [pan addTarget:self action:@selector(handleRecognizer:)];
    
}

- (void)handleRecognizer:(UIPanGestureRecognizer *)recognizer{

    switch (recognizer.state) {
        case UIGestureRecognizerStateBegan:
            self.interacting = YES;
            [self moveBackWithTransitionType:_currentType];
            break;
        case UIGestureRecognizerStateChanged:
            [self handleTransitionWithType:_currentType andRecognizer:recognizer];
            break;
        case UIGestureRecognizerStateEnded:
        case UIGestureRecognizerStateCancelled:
        {
            self.interacting = NO;
            if (self.canEndTransition)
            {
                [self finishInteractiveTransition];
            }
            else if (!self.canEndTransition || recognizer.state == UIGestureRecognizerStateCancelled)
            {
                [self cancelInteractiveTransition];
            }
        }
            break;
        default:
            break;
    }
    
}

- (void)moveBackWithTransitionType:(SPTransitionType)type{

    switch (type) {
        case SPTransitionTypeModal:
            [self.presentingController dismissViewControllerAnimated:YES completion:nil];
            break;
        case SPTransitionTypePush:
            [self.presentingController.navigationController popViewControllerAnimated:YES];
            break;
    }
    
}

- (void)handleTransitionWithType:(SPTransitionType)type andRecognizer:(UIPanGestureRecognizer *)recognizer{

    CGRect screenBounds = [UIScreen mainScreen].bounds;
    CGPoint point = [recognizer translationInView:recognizer.view.superview];// translationInView
    
    switch (type) {
        case SPTransitionTypePush:
        {
            CGFloat percent = point.x/(screenBounds.size.width /2);// 有效距离为屏幕宽度的一半
            percent = fminf(fmax(percent, 0.0), 1.0);
            self.canEndTransition = percent > 0.5;// 活动超过屏幕宽度的1/4则可完成转场
            [self updateInteractiveTransition:percent];
        }
            break;
        case SPTransitionTypeModal:
        {
            CGFloat actionPoint = point.y/(screenBounds.size.height /2);// 有效距离为屏幕高度的一半
            actionPoint = fminf(fmaxf(actionPoint, 0.0), 1.0);
            self.canEndTransition = actionPoint > 0.5;// 滑动超过屏幕高度的1/4则可完成转场
            [self updateInteractiveTransition:actionPoint];
        }
            break;
    }
    
}

- (CGFloat)completionSpeed{

    return 1 - self.percentComplete;
    
}

@end
```
因为加入了手势处理，所以我们需要将Pop中上报转场完成的地方稍加改动
考虑手势取消转场的情况，取消时，将fromVC的内容再次显示即可
```
// 开始动画
    [UIView animateWithDuration:[self transitionDuration:transitionContext] delay:0 usingSpringWithDamping:0.6 initialSpringVelocity:1/0.6 options:UIViewAnimationOptionCurveEaseInOut animations:^{
        
        snapView.frame = targetFrame;
        snapView.alpha = 0;
        toVC.view.alpha = 1;
        
    } completion:^(BOOL finished) {
        
        BOOL completeTransiton = ![transitionContext transitionWasCancelled];
        snapView.hidden = YES;
        
        if (!completeTransiton) fromVC.showContent = YES;
        
        [snapView removeFromSuperview];
        [transitionContext completeTransition:completeTransiton];
        
    }];
```
看一下我们的最终效果

![swipeTrans.gif](http://upload-images.jianshu.io/upload_images/1742463-3d4708e7ba56863f.gif?imageMogr2/auto-orient/strip)

---
### 总结
** 1.理解ContainerView的作用 **
** 2.务必上报transition结束或取消状态 **
** 3.区分`UIViewControllerTransitioningDelegate`、`UINavigationControllerDelegate`的使用场景 **
** 4.转场动画的关键，在于处理fromVC和toVC的view的变化，最终还是回归到UIView动画 **
** 5.体会自定义transition时，代理方法带来的简洁高效 **

