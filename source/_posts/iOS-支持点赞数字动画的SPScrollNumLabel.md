---
title: iOS-支持点赞数字动画的SPScrollNumLabel
date: 2017-11-22 13:44:08
tags: "iOS"
categories: "iOS"
---

简单实现支持点赞数字动画的SPScrollNumLabel

<!--more-->

[Github:SPScrollNumLabel](https://github.com/Tr2e/SPScrollNumLabel)
**如果你觉得帮助到你了，希望给鶸一个Star，谢谢**
[简书](http://www.jianshu.com/u/737f895deaec)
**如果你发现任何问题，请一定要给我拍砖，谢谢**

**项目效果**
![项目应用效果](https://github.com/Tr2e/SPScrollNumLabel/raw/master/Picture/timelineDemo.gif)

**Ver 0.0.1**
![Demo效果](https://github.com/Tr2e/SPScrollNumLabel/raw/master/Picture/Demo.gif)

**Ver 0.0.2**
![Text效果](https://github.com/Tr2e/SPScrollNumLabel/raw/master/Picture/text.gif)

## Api

```
@property (nonatomic, assign) IBInspectable NSInteger targetNumber;// default is 0
@property (nonatomic, assign) IBInspectable CGFloat animateDuration;// default is 0.25
@property (nonatomic, assign) IBInspectable BOOL isCommonLabel;// default is NO
@property (nonatomic, assign) BOOL centerPointPriority;// default is NO
- (void)increaseNumber:(NSInteger)increasedNum;
- (void)decreaseNumber:(NSInteger)decreasedNum;
```

* 设置目标数字
* 设置数字滚动的动画时间
* 设置为普通的`UILabel`使用
* 设置是否为`center`属性优先布局，针对只设置`foo.center`的情况，详情见Demo
* 数字增加方法
* 数字减少方法

## 设置

1. **对文字及字体颜色等常见参数的设置，直接通过UILabel的参数设置即可.**
2. **输入数字支持targetNum及text两种属性输入**
3. **targetNumber及text的输入，都请放在字体属性设置完成后**

*注意点：如果输入的文字为中文，请不要设置Label的backgroundColor属性，否则无法正常显示*

### 纯代码

1. 设置frame时,**如果size属性的宽不能适应展示宽度，都会自动调整,如果size属性的高度不能容纳展示高度，会自动调整，能容纳则不做任何处理**
2. `targetNumber`的赋值，请务必放在配置参数的**最后**

```
CGRect screenBounds = [UIScreen mainScreen].bounds;
SPScrollNumLabel *num = [[SPScrollNumLabel alloc] initWithFrame:(CGRect){CGPointMake(screenBounds.size.width/2 - 50, 100),CGSizeMake(2, 100)}];
    
    // 设置frame时
    // 如果size属性的宽不能适应展示宽度，都会自动调整
    // 如果size属性的高度不能容纳展示高度，会自动调整，能容纳则不做任何处理
    // num.frame = (CGRect){CGPointMake(screenBounds.size.width/2 - 50, 100),CGSizeMake(2, 100)};
    
// 字体属性，直接赋值
num.textColor = [[UIColor whiteColor] colorWithAlphaComponent:0.5];
num.font = [UIFont systemFontOfSize:40 weight:UIFontWeightBold];
num.backgroundColor = [[UIColor purpleColor] colorWithAlphaComponent:0.4];
    
    
// 如果采用center赋值 需要设置是否中心点优先
SPScrollNumLabel *centerLabel = [[SPScrollNumLabel alloc] init];
centerLabel.center = CGPointMake(screenBounds.size.width/2, 250);
centerLabel.centerPointPriority = YES;
centerLabel.textColor = [[UIColor whiteColor] colorWithAlphaComponent:0.5];
centerLabel.font = [UIFont systemFontOfSize:35 weight:UIFontWeightThin];
centerLabel.backgroundColor = [[UIColor blueColor] colorWithAlphaComponent:0.4];
// 属性配置完成后，赋值 默认为0
centerLabel.text = @"998";
    
// 如果想当做普通的UILabel用 比如特殊值"1千"等 打开isCommonLabel 直接按照UILabel的使用即可
SPScrollNumLabel *commonLabel = [[SPScrollNumLabel alloc] init];
commonLabel.isCommonLabel = YES;
commonLabel.textColor = [[UIColor whiteColor] colorWithAlphaComponent:0.5];
commonLabel.font = [UIFont systemFontOfSize:35 weight:UIFontWeightThin];
commonLabel.backgroundColor = [[UIColor blueColor] colorWithAlphaComponent:0.4];
commonLabel.text = @"我可以当普通label用哦";
[commonLabel sizeToFit];
commonLabel.center = CGPointMake(screenBounds.size.width/2, commonLabel.frame.size.height/2+34);
```

### xib

xib支持直接设置，你可以将必要的参数在这里直接设置：**颜色**、**字体**、**动画时间**、**是否是个普通Label**、**动画翻转时间**
![demo 1](https://github.com/Tr2e/SPScrollNumLabel/raw/master/Picture/Snip20171122_1.png)
![demo 2](https://github.com/Tr2e/SPScrollNumLabel/raw/master/Picture/Snip20171122_3.png)

## 如何应用

使用**pod**或者直接拖拽相应文件夹到你的工程下
```
pod 'SPScrollNumLabel' ,'~> 0.0.2'
```

**20171124 Ver.0.0.2**:支持`label.text`的直接赋值

**20171123 Ver.0.0.1**

---

**Enjoy It**


