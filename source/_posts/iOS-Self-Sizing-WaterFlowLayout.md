---
title: iOS-Autolayout自动计算itemSize的UICollectionViewLayout瀑布流布局
date: 2017-09-21 11:51:27
tags: 'iOS'
categories: 'iOS'
---

### 前言

探究`Custom Layout`及应用实践

<!--more-->

### UICollectionViewLayout基础知识
#### **Custom Layout**
> **官方描述**
> An abstract base class for generating layout information for a collection view
> The job of a layout object is to determine the placement of cells, supplementary views, and decoration views inside the collection view’s bounds and to report that information to the collection view when asked
> 
> [官方文档](https://developer.apple.com/documentation/uikit/uicollectionviewlayout)

`UICollectionViewLayout`的功能为向`UICollectionView`提供布局信息，不仅包括`cell`的**布局信息**，也包括**追加视图**和**装饰视图**的布局信息。实现一个自定义**`Custom Layout`**的常规做法是继承`UICollectionViewLayout`类

#### **重载的方法**
* `prepareLayout`:**准备布局属性**
* `layoutAttributesForElementsInRect:`**返回rect中的所有的元素的布局属性**`UICollectionViewLayoutAttributes`可以是**cell**，**追加视图**或**装饰视图**的信息，通过不同的`UICollectionViewLayoutAttributes`初始化方法可以得到不同类型的`UICollectionViewLayoutAttributes`
> * `layoutAttributesForCellWithIndexPath:`
>* `layoutAttributesForSupplementaryViewOfKind:withIndexPath:`
>* `layoutAttributesForDecorationViewOfKind:withIndexPath:`
* `collectionViewContentSize`**返回contentSize**

#### 执行顺序
1. `-(void)prepareLayout`将被调用，默认下该方法什么没做，但是在自己的子类实现中，一般在该方法中**设定一些必要的`layout的`结构和初始需要的参数等**
2. `-(CGSize) collectionViewContentSize`将被调用，以确定`collection`应该占据的尺寸。注意这里的尺寸不是指可视部分的尺寸，而应该是所有内容所占的尺寸。**`collectionView`的本质是一个`scrollView`，因此需要这个尺寸来配置滚动行为**
3. `-(NSArray *)layoutAttributesForElementsInRect:(CGRect)rect`

### 引入AutoLayout自动计算的瀑布流
#### 关于瀑布流
网上前辈们已经写烂了，这里只简述：
* `-(void)prepareLayout`中:就是通过一个记录列高度的**数组**(或**字典**)，在创建`LayoutAttributes`的frame时确定当前最短列，根据外部传入的相关的`spacing`及`collectionView`的`inset`属性，确定**宽度**、**frame**等信息，存入**Attributes**的数组。
* `-(CGSize) collectionViewContentSize`中：通过**列高度数组**很容易确定当前范围，**contentSize不等于collectionview的bounds.size**,计算时留意一下
* `-(NSArray *)layoutAttributesForElementsInRect:(CGRect)rect`：返回第一步中计算获得的**Attributes数组**即可

以上可以帮助我们实现一个瀑布流的效果，但是离实际应用还有一段差距。

**分析：**
实际应用中，我们的网络请求是会有一个**pageSize**的，而且列表的赋值通常是直接进行数据源的赋值然后`reloadData`。所以数据源个数等于**pageSize**时，我们认为是刷新，大于时，则为分页加载。
根据这套逻辑，这里将**pageSize**及**dataSource**作为属性引入到`Custom Layout`中，同时维护一个记录计算结果的数组**itemSizeArray**，提高计算效率，具体代码如下：
```
- (void)calculateAttributesWithItemWidth:(CGFloat)itemWidth{
    BOOL isRefresh = self.datas.count <= self.pageSize;
    if (isRefresh) {
        [self refreshLayoutCache];
    }
    NSInteger cacheCount = self.itemSizeArray.count;
    for (NSInteger i = cacheCount; i < self.datas.count; i ++) {
        CGSize itemSize = [self calculateItemSizeWithIndex:i];
        UICollectionViewLayoutAttributes *layoutAttributes = [self createLayoutAttributesWithItemSize:itemSize index:i];
        [self.itemSizeArray addObject:[NSValue valueWithCGSize:itemSize]];
        [self.layoutAttributesArray addObject:layoutAttributes];
    }
}
```
```
- (UICollectionViewLayoutAttributes *)createLayoutAttributesWithItemSize:(CGSize)itemSize index:(NSInteger)index{
    UICollectionViewLayoutAttributes *layoutAttributes = [UICollectionViewLayoutAttributes layoutAttributesForCellWithIndexPath:[NSIndexPath indexPathForItem:index inSection:0]];
    struct SPColumnInfo shortestInfo = [self shortestColumn:self.columnHeightArray];
    // x
    CGFloat itemX = (self.itemWidth + self.interitemSpacing) * shortestInfo.columnNumber;
    // y
    CGFloat itemY = self.columnHeightArray[shortestInfo.columnNumber].floatValue + self.lineSpacing;
    // size
    layoutAttributes.frame = (CGRect){CGPointMake(itemX, itemY),itemSize};
    self.columnHeightArray[shortestInfo.columnNumber] = @(CGRectGetMaxY(layoutAttributes.frame));
    return layoutAttributes;
}
```
```
- (void)refreshLayoutCache{
    [self.layoutAttributesArray removeAllObjects];
    [self.columnHeightArray removeAllObjects];
    [self.itemSizeArray removeAllObjects];
    for (NSInteger index = 0; index < self.columnNumber; index ++) {
        [self.columnHeightArray addObject:@(self.viewInset.top)];
    }
}
```
代码里可以看到，`itemSizeArray`的属性，用于记录自动计算的`itemSize`，通过这个属性可以帮助我们**减少不必要的重复计算**
#### 关于自动计算

![](http://upload-images.jianshu.io/upload_images/1742463-8c4c4a64292c2911.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**注意**：
* **Self-size**要求我们的约束自上而下设置，确保能够通过**Constraint**计算获得准确的高度。具体不再赘述
* **本Demo仅适用图片比例确定的瀑布流**，如果需求是图片size自适应，需要服务器返回能够计算的必要参数

自动计算的思路，**类似[UITableView-FDTemplateLayoutCell](https://github.com/forkingdog/UITableView-FDTemplateLayoutCell)**，通过**xibName**或**className**初始化一个**template cell**，**注入数据**并添加**横向约束**后，利用**`systemLayoutSizeFittingSize`**方法获取系统计算的高度后，移除添加的**横向约束**（***其中有个iOS10.2后的约束计算变化，需要我们手动对`cell.contentView`添加四周的约束，AutoLayout才能准确计算高度。请注意代码中对系统判断的一步***）

这里我们为`UICollectionViewCell`添加了一个`Category`，用于统一数据的传入方式
```
#import <UIKit/UIKit.h>

@interface UICollectionViewCell (FeedData)
@property (nonatomic, strong) id feedData;
@property (nonatomic, strong) id subfeedData;
@end

// --------------------------------------

#import "UICollectionViewCell+FeedData.h"
#import <objc/runtime.h>

static NSString *AssociateKeyFeedData = @"AssociateKeyFeedData";
static NSString *AssociateKeySubFeedData = @"AssociateKeySubFeedData";
@implementation UICollectionViewCell (FeedData)
@dynamic feedData;
@dynamic subfeedData;

- (void)setFeedData:(id)feedData{
    objc_setAssociatedObject(self, &AssociateKeyFeedData, feedData, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (id)feedData{
    return objc_getAssociatedObject(self, &AssociateKeyFeedData);
}

- (void)setSubfeedData:(id)subfeedData{
    objc_setAssociatedObject(self, &AssociateKeySubFeedData, subfeedData, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (id)subfeedData{
    return objc_getAssociatedObject(self, &AssociateKeySubFeedData);
}

@end

```
关键代码如下：
* itemSize
```
- (CGSize)calculateItemSizeWithIndex:(NSInteger)index{
    NSAssert(index < self.datas.count, @"index is incorrect");
    UICollectionViewCell *tempCell = [self templateCellWithReuseIdentifier:self.reuseIdentifier withIndex:index];
    tempCell.feedData = self.datas[index];
    CGFloat cellHeight = [self systemCalculateHeightForTemplateCell:tempCell];
    return CGSizeMake(self.itemWidth, cellHeight);
}
```
* 获取一个计算使用的**Template Cell**，保存避免重复提取
```
- (UICollectionViewCell *)templateCellwithIndex:(NSInteger)index{
    if (!self.templateCell) {
        if (self.className) {
            Class cellClass = NSClassFromString(self.className);
            UICollectionViewCell *templateCell = [[cellClass alloc] init];
            self.templateCell = templateCell;
        }else if (self.xibName){
            UICollectionViewCell *templateCell = [[NSBundle mainBundle] loadNibNamed:self.xibName owner:nil options:nil].lastObject;
            self.templateCell = templateCell;
        }
    }
    return self.templateCell;
}
```
* AutoLayout Self-sizing
```
- (CGFloat)systemCalculateHeightForTemplateCell:(UICollectionViewCell *)cell{
    CGFloat calculateHeight = 0;
    
    NSLayoutConstraint *widthForceConstant = [NSLayoutConstraint constraintWithItem:cell.contentView attribute:NSLayoutAttributeWidth relatedBy:NSLayoutRelationEqual toItem:nil attribute:NSLayoutAttributeNotAnAttribute multiplier:1.0 constant:self.itemWidth];

    static BOOL isSystemVersionEqualOrGreaterThen10_2 = NO;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        isSystemVersionEqualOrGreaterThen10_2 = [UIDevice.currentDevice.systemVersion compare:@"10.2" options:NSNumericSearch] != NSOrderedAscending;
    });
    
    NSArray<NSLayoutConstraint *> *edgeConstraints;
    if (isSystemVersionEqualOrGreaterThen10_2) {
        // To avoid conflicts, make width constraint softer than required (1000)
        widthForceConstant.priority = UILayoutPriorityRequired - 1;

        // Build edge constraints
        NSLayoutConstraint *leftConstraint = [NSLayoutConstraint constraintWithItem:cell.contentView attribute:NSLayoutAttributeLeft relatedBy:NSLayoutRelationEqual toItem:cell attribute:NSLayoutAttributeLeft multiplier:1.0 constant:0];
        NSLayoutConstraint *rightConstraint = [NSLayoutConstraint constraintWithItem:cell.contentView attribute:NSLayoutAttributeRight relatedBy:NSLayoutRelationEqual toItem:cell attribute:NSLayoutAttributeRight multiplier:1.0 constant:0];
        NSLayoutConstraint *topConstraint = [NSLayoutConstraint constraintWithItem:cell.contentView attribute:NSLayoutAttributeTop relatedBy:NSLayoutRelationEqual toItem:cell attribute:NSLayoutAttributeTop multiplier:1.0 constant:0];
        NSLayoutConstraint *bottomConstraint = [NSLayoutConstraint constraintWithItem:cell.contentView attribute:NSLayoutAttributeBottom relatedBy:NSLayoutRelationEqual toItem:cell attribute:NSLayoutAttributeBottom multiplier:1.0 constant:0];
        edgeConstraints = @[leftConstraint, rightConstraint, topConstraint, bottomConstraint];
        [cell addConstraints:edgeConstraints];
    }
    
    // system calculate
    [cell.contentView addConstraint:widthForceConstant];
    calculateHeight = [cell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize].height;
    // clear constraint
    [cell.contentView removeConstraint:widthForceConstant];
    if (isSystemVersionEqualOrGreaterThen10_2) {
        [cell removeConstraints:edgeConstraints];
    }
    return calculateHeight;
}
```
#### 如何使用
* 初始化时对所有必要属性进行赋值
```
    SPWaterFlowLayout *flowlayout = [[SPWaterFlowLayout alloc] init];
    flowlayout.columnNumber = 2;
    flowlayout.interitemSpacing = 10;
    flowlayout.lineSpacing = 10;
    flowlayout.pageSize = 54;
    flowlayout.xibName = @"TestView";
    UICollectionView *test = [[UICollectionView alloc] initWithFrame:self.view.bounds collectionViewLayout:flowlayout];
    test.contentInset = UIEdgeInsetsMake(10, 10, 5, 10);
    [self.view addSubview:test];
    test.delegate = self;
    test.dataSource = self;
    [test registerNib:[UINib nibWithNibName:@"TestView" bundle:nil] forCellWithReuseIdentifier:@"Cell"];
    test.backgroundColor = [UIColor whiteColor];
```
* Refresh及LoadMore中更新`dataSource`

**Refresh**
```
test.refreshDataCallBack = ^{
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            self.pageTag = 0;
            NSArray *datas = [SPProductModel productWithIndex:0];
            flowlayout.datas = datas;
            wtest.sp_datas = [datas mutableCopy];
            [wtest doneLoadDatas];
            [wtest reloadData];
        });
    };
```
**LoadMore**
```
 test.loadMoreDataCallBack = ^{
        self.pageTag ++;
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            NSArray *datas = [SPProductModel productWithIndex:self.pageTag];
            NSArray *total = [flowlayout.datas arrayByAddingObjectsFromArray:datas];
            flowlayout.datas = total;
            wtest.sp_datas = [total mutableCopy];
            [wtest doneLoadDatas];
            [wtest reloadData];
        });
    };
```
#### 效果
题外话：iPhone X让我们除了**64**，又记住了**88和812**，自己写**Refresh**的朋友，记得**更新下机型判断**
![waterflow.gif](http://upload-images.jianshu.io/upload_images/1742463-6944255e81212bf9.gif?imageMogr2/auto-orient/strip)
#### Demo地址
[SPWaterFlowLayout](https://github.com/Tr2e/SPWaterFlowLayout)
[笔者简书地址](http://www.jianshu.com/p/bd856b5e140a)
![](http://upload-images.jianshu.io/upload_images/1742463-4181952cb01a5b2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

