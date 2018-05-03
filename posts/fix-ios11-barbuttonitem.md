---
layout: post
title: "iOS 11 怎样为导航条上的 UIBarButtonItem 设置间距"
date: 2018-01-06 14:28:20 +0800
comments: true
categories: 
---

以前我们常用 `fixedSpace` 的方式为 UINavigationBar 上的 UIBarButtonItem 设置间距：

```objc
UIBarButtonItem *negativeSpacer = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemFixedSpace 
                                                                                target:nil 
                                                                                action:nil];

negativeSpacer.width = -8;

[self.navigationItem setLeftBarButtonItems:@[negativeSpacer, button] animated:NO];
```

<!--more-->

然而在 iOS 11 下 `UIBarButtonItem` width 属性不但失效了，`UIBarButtonItem` 也开始使用 auto layout 布局，对此我们需要设置 `UIBarButtonItem` 子 view 的约束。除此之外，苹果还修改了 `UINavigationBar` 的实现。直到 iOS 10 `UINavigationBar` 都是采用手动布局，所有的子 view 都是直接加在 `UINavigationBar` 上。但是，从 iOS 11 开始， `UINavigationBar` 使用了 auto layout 来布局它的内容子 view，并且子 view 加在了 `_UINavigationBarContentView` 上。

先来看看 iOS 11 下 `UINavigationBar` 的视图层级：

```
UINavigationBar
      | _UIBarBackground
      |    | UIImageView
      |    | UIImageView
      | _UINavigationBarLargeTitleView
      |    | UILabel
      | _UINavigationBarContentView 
      |    | _UIButtonBarStackView
      |    |    | _UIButtonBarButton
      |    |    |    | _UIModernBarButton
      |    |    |    |    | UIButtonLabel
      | _UINavigationBarContentView
      |    | _UIButtonBarStackView
      |    |    | _UITAMICAdaptorView // customView
      |    |    |    | UIBarButtonItem
      |    |    |    |    | UIImageView
      |    |    |    |    | UIButtonLabel
```

通过 View Debug 工具可知，原来是 stackView 限制了 customView 的宽度以及引起了偏移：

``` 
contentView |<-fullWidth----------->|
stackView     |<-stackViewWidth->|
customView    |<-reducedWidth--->|
```

在此次深挖之前，贝聊客户端的开发小哥们由于项目工期紧以及适配 iOS 11 工作量大，暂时是通过设置 `UIButton` 的 `setContentEdgeInsets:` 来实现的，这在当时看来是以最小的改动完成了适配，直到 iOS 11.2 这个版本的推出，我们发现当侧滑返回时，导航条的返回按钮会被切掉一点角。（这个方法还有个小缺点是点击区域太小了）

![img1][image-1]

碰巧的是，我的 leader 恰好发现了钉钉也有类似的问题。

![][image-2]

iOS 11 虽然已经推出好几个月了，这个问题可能还在困扰着部分同行，接下来就讲讲贝聊是如何解决这个问题的。

由于大家知道 fixed space 失效是系统换成了 auto layout 来实现，所以网上的大部分文章也都是修改 constraint。遗憾的是，我谷歌到挺多使用这种方式去修改要获取到 `UINavigationBar` 的私有子 view，譬如 `contentView` 或 `barStackView`，再为私有子 view 添加 leading 和 trailing 的约束等。

我并没有尝试这种方法的可行性，因为始终觉得获取私有子 view 的做法比较脆弱，苹果一旦更换实现，程序的健壮性不好保障。但可以确定的是，解决这个问题的思路大致是修改约束，设法摆脱 stackView 的限制。

在 auto layout 中，约束是使用 alignment rectangle 来确定视图的大小和位置。先看看 alignment rectangle 的作用是怎样，下图摘自《iOS Auto Layout Demystified》：

![img2][image-3]

![img3][image-4]

书中对此的说明是，假如设计师给了你张带角标的气泡图片，程序只期望对气泡进行居中，而图片的 frame 却包含了角标部分，这时可以 override `alignmentRectForFrame:`、`frameForAlignmentRect`。`UIView` 也给出了相对简便的属性 `alignmentRectInsets`，需要注意的是，一旦设置了 `alignmentRectInsets`，view 的 frame 就会根据 alignment rectangle 和 `alignmentRectInsets` 计算出来。

有了以上的概念后，贝聊的修复方法是子类化一个 UIBarButtonItem 的 customView：

```objc
@interface BLNavigationItemCustomView: UIView

@property (nonatomic) UIEdgeInsets alignmentRectInsetsOverride;

@end

@implementation BLNavigationItemCustomView

- (UIEdgeInsets)alignmentRectInsets {
    if (UIEdgeInsetsEqualToEdgeInsets(self.alignmentRectInsetsOverride, UIEdgeInsetsZero)) {
        return super.alignmentRectInsets;
    } else {
        return self.alignmentRectInsetsOverride;
    }
    
}

@end
```

再就是创建 customView 时针对 iOS 11 做特殊处理，以返回按钮为例：

```objc
if (@available(iOS 11.0, *)) {
    button.alignmentRectInsetsOverride = UIEdgeInsetsMake(0, offset, 0, -(offset));
    button.translatesAutoresizingMaskIntoConstraints = NO;
    [button.widthAnchor constraintEqualToConstant:buttonWidth].active = YES;
    [button.heightAnchor constraintEqualToConstant:44].active = YES;
                    
}
```

之所以设置 widthAnchor、heightAnchor 是前文提到的需要对 `UIBarButtonItem` 子 view 设置约束，我在实现时就遇到了怎么修改 frame 都无法撑大 customView 的情况，后来发现是没设置 widthAnchor。我们接着用 View Debug 来看看实现的效果：

![img4][image-5]

这儿有个问题就是 customView 有小部分超出了 stackView 的 bounds，导致了超出部分无法接收点击。有趣的是，使用 iOS 11 之前 fixed space 添加间距的做法可以减少 stackView 的 margin。

```objc
UIBarButtonItem *spacer = [UIBarButtonItem bl_barButtonItemSpacerWithWidth:-(offset)];
self.navigationItem.leftBarButtonItems = @[spacer, barButtonItem];
```

结合上 fixed space 和 alignmentRectInsets，customView 将不再超出它的父视图：

![img5][image-6]

总之，我们需继承复写 ` alignmentRectInsets` 的 `BLNavigationItemCustomView `，然后继续保持之前版本 fixed space 的处理，只针对 iOS 11 为 customView 新增约束，就可使间距问题在新旧系统得以解决。

### 总结
不客气的说，iOS 11 真的是一个挺难适配的版本，期间我都差点放弃对导航条间隔的适配了，好在最后还是顺利解决了。如果你有更好的方式解决，欢迎赐教。

[image-1]:	https://user-images.githubusercontent.com/5633917/34596594-044690be-f21c-11e7-8000-e045dd1f0822.png
[image-2]:	https://user-images.githubusercontent.com/5633917/34637834-7982edb8-f2f9-11e7-84f5-9fef5306e910.JPG
[image-3]:	https://user-images.githubusercontent.com/5633917/34598265-97cfd8e0-f226-11e7-886f-3d0d2f0d79ce.jpeg
[image-4]:	https://user-images.githubusercontent.com/5633917/34598273-b0486090-f226-11e7-998a-9d695a1f4bbc.jpeg
[image-5]:	https://user-images.githubusercontent.com/5633917/34602108-8b91454c-f239-11e7-9677-9c8434ec0d8a.jpeg
[image-6]:	https://user-images.githubusercontent.com/5633917/34602516-04f734b8-f23b-11e7-809a-5574dc5f6b3e.jpeg