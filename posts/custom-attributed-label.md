#如何实现一个 AttributedLabel

Core Text 是苹果提供的富文本排版技术，可以定制开发图文混排功能，DTCoreText、Nimbus、YYLabel 等优秀的开源库底层都是基于 Core Text 的封装和扩展。本文将介绍 Core Text 的基本用法，逐步讲解我是如何封装一个 [AttributedLabel][1] 的。


### 文本排版简述
文本排版是根据给定的文本（text）、字体（font）、绘制区域（shape）、行高（line height）等相关属性，生成出字形（glyphs）布局在屏幕绘制区的适当位置。排版的核心就是将字符（characters）转换成字形，将字形排列成行（lines），再将行排成段落（paragraphs）。用代码表达就是下边寥寥几行。

```objc
    CTFramesetterRef framesetter = CTFramesetterCreateWithAttributedString((CFAttributedStringRef)attributedString);
    CTFrameRef frame = CTFramesetterCreateFrame(framesetter, CFRangeMake(0, [attributedString length]), path, NULL);
    CTFrameDraw(frame, context);
```

这里的主要步骤有：

1. 创建 Attributed String；
2. 创建 CTFramesetter，这是 Core Text 排版的核心类，它会贯穿整个排版过程；
3. 创建 CGPath，即绘制的区域；
4. 通过 CTFramesetter 和 CGPath 创建 CTFrame，然后可将其绘制在当前的 context 上；
5. 别忘了调用 CFRelease 释放对象。

在继续深入代码之前，先了解以下几个小概念：

#### 字形与字体
简单说字体就是映射到字符的字形集合，以下就是字符 a （ascii 码为 97）的不同字形：

<img src="https://user-images.githubusercontent.com/5633917/33275706-6f4178aa-d3ce-11e7-921c-4afd1bb6b76a.png" width="200"/>

而同一字体下字形也可能会有所不同，在英文中比较常见的如连字，典型的就是fi中 i 的点常与 f 的钩合并：

<img src="https://user-images.githubusercontent.com/5633917/33275829-b2a349ac-d3ce-11e7-931a-41dcd8c2f3ec.png" width="400"/>

接下来说说字体，在开发中我们常说同一字体不同字号，比如  `[UIFont systemFontOfSize: 16]` 和 `[UIFont systemFontOfSize: 18]`，或者同一字体但是加粗显示，又如  `[UIFont systemFontOfSize: 16]` 和 `[UIFont boldSystemFontOfSize: 16]`，又或者斜体，然而这对于系统而言是完全不同的字体。这儿想说明的是：不同字号是不同的字体，粗体相对普通也是不同的字体，而给文本添加下划线却是个例外（下划线是系统额外画的一条装饰线）。

有时我们在开发中也会接触到字体的 Ascent 和 Descent，其实就是在于字形度量（Glyph metrics）打交道：

<img src="https://user-images.githubusercontent.com/5633917/33275873-d1a33c5e-d3ce-11e7-985d-5b8e7bc8b9e1.png" width="600"/>

由上图可知，一个字符最高点到基线的偏移叫做 Ascent，最低点到基线的偏移叫做 Descent，单行的行高 Line Height 由 Ascent、Descent 与 Line Gap 相加得出。

#### 文本的绘制

<img src="https://developer.apple.com/library/content/documentation/StringsTextFonts/Conceptual/TextAndWebiPhoneOS/Art/core_text_arch_2x.png" width="400"/>

Core Text 需要使用 CTFramesetter 对文本进行布局，位于上图中最顶端的 CTFramesetter，它要求以 Attributed String 和绘制区域的形状（CGPath）作为入参，来创建 CTFrame（可以不止一个 CTFrame） ，顾名思义，这就是文本布局所在的 frame，确定好绘制区域后，framesetter 就能将段落样式（NSParagraphStyle）的 lineBreakMode、lineSpacing 等属性应用于此。
这里有必要提一下 CTRun，从 CTRun 我们可以获取许多重要的属性，这在开发排版功能的时候非常有用，下面这张图有助于我们了解什么是 CTRun：

<img src="https://user-images.githubusercontent.com/5633917/33275976-20fb6826-d3cf-11e7-9add-089329fb271d.png" width="400"/>

这一行文本可以认为是一个 CTLine 对象，由从左往右的顺序依次包含了默认字体样式、加粗字体样式、默认字体样式、小字号蓝色样式、正常字号蓝色样式和默认字体样式共 6 种 Attributed。每一种样式的字符则表示一个 CTRun 对象。

了解了这些概念之后，就可以实现排版功能了。

### 实现一个简单的 AttributedLabel

进入正题之前，再储备些基础知识。

##### Core Foundation 内存管理规则

Core Text 使用了 Core Foundation 基于 C 语言的 API，所以需要遵循 Core Foundation 的内存管理规则。

- 创建方法名中含有 “Create” 或 “Copy”，需要调用 CFRelease 释放内存

```objc
CTFramesetterRef CTFramesetterCreateWithAttributedString(
CFAttributedStringRef string )
```
- 返回 CF 对象方法名中不含 “Create” 和 “Copy”，无需手动释放内存

```objc
CFStringRef CFAttributedStringGetString(CFAttributedStringRef aStr)
```

明白了这点，就对项目中什么时候该调用 CFRelease，什么时候不该调用做到心中有数了。

关于 `__bridge` 关键字

- `__bridge` 只是声明类型转变，但不做内存管理规则的转变。
- `__bridge_retained` 表示指针类型转变的同时，将内存管理由原来的 Objective-C 交给 Core Foundation 处理，即 ARC to MRC。
- `__bridge_transfer` 表示内存管理由 Core Foundation 交给 Objective-C，即 MRC to ARC。

##### 关于坐标系
另外，Core Text 最初是设计给 mac 的，它的坐标系是 mac 坐标系（原点在左下角），所以通常需要对坐标进行翻转，这也是下文提及为什么需要翻转的缘由。

```objc
CGContextSetTextMatrix(context, CGAffineTransformIdentity);
CGContextTranslateCTM(context, 0, size.height);
CGContextScaleCTM(context, 1.0, -1.0);
```

##### 借助下面这张类关系图让我们直奔主题。

![][image-6]

#### 1.堪当重任的 CALayer
相对于 UIView，CALayer 通常是比较“轻”的，我们在日常开发中接触 layer 比较多的还是设置 cornerRadius、contents、mask 或者做个动画等，而在这个项目中，依靠 layer 的 `- (void)display` 方法，让其充当了一个 “桥梁” 的作用。

先来了解下 `- (void)display` 方法，如文档里所说，layer 会在适当的时候调用该方法来更新 layer 的 contents，但是并不建议直接调用该方法，子类化可以重写该方法，并能直接设置 layer 的 contents。文档的最后一句话大大盘活了自定义的 AttributedLabel，当 AttributedLabel 需要改变 text、frame、font、attributedString…时，AttributedLabel 不用关心具体的绘制，只需告知下 layer 需要 display 即可。由于将 AttributedLabel 的 `+ (Class)layerClass` 返回了子类化的 layer。

```objc
+ (Class)layerClass {
    return [ZPLabelLayer class];
}
```

layer 的 delegate 对象就是 AttributedLabel，所以 layer 就能通过它的 delegate 属性获取到 AttributedLabel 的上述属性，进一步调用 Core Text 绘制出新的 contents 进行设置。这是做这个项目时最干净利落的一个地方。
 
#### 2.文本高亮交互的处理
如果无需处理高亮交互等定制（截断、附件）效果，我们在拿到 NSAttributedString 和 CGPath 即可将文本绘制到 context 上。对于链接而言，虽然我们能通过 NSDataDetector 标记出文本中哪些地方需要高亮显示，但是需求往往要能对链接进行点击跳转，在使用 CTFrameDraw 方法绘制文本时，既不知道高亮过的文本位置，更无法谈及对高亮文本的交互响应了。

幸运的是，Core Text 另外还有个稍微复杂点的绘制方法 CTLineDraw，从名字可以得知它是用来绘制 line 的，感观上要比 CTFrameDraw 的确要精细许多。我们先看看添加高亮功能的实现思路。

- 能响应点击的回调 block
- 接受高亮颜色、range、backgroundColor 等

假设上述高亮相关属性都由 AttributedLabel 处理，使用者每次添加高亮不仅要让 AttributedLabel 改变内部文本的 Attributed 属性，考虑到一段文本可能有多处高亮，其本身也还需要维护一个处理高亮的数组。然而对设置高亮来说，这本就是 NSAttributedString 能做到的事，若让 UI 层来处理这些逻辑并不是很好。再者，对于调用者来说，虽然可以将上述属性封装成 model 方便 AttributedLabel 使用，但如果想复用 NSAttributedString 就变得不可能了。看来交由 NSAttributedString 来处理高亮相关属性是最合适不过的了，这里通过创建 NSAttributedString 的 category 和  AssociatedObject 满足了需求。

最终从 NSAttributedString 中获取到高亮的 ranges，再配合 CTLineDraw 绘制行的时候获取到 run （文章前面介绍）的 range，先来看看代码：

```objc
self.ranges = attributedStr.highlightRangeArray;  // 获取 ranges
...
// 遍历行
for (CFIndex lineIndex = 0; lineIndex < numberOfLines; lineIndex++) {
        CTLineRef line = CFArrayGetValueAtIndex(lines, lineIndex);
    ...
    CFArrayRef runs = CTLineGetGlyphRuns(line);
    // 遍历行的每一个 run
        for (int j = 0; j < CFArrayGetCount(runs); j++) {
        ...
        CFRange range = CTRunGetStringRange(run);
            
             for (NSString *rangeString in self.ranges) {
                    NSRange hightlightRange = NSRangeFromString(rangeString);
                    NSRange lineRange = NSMakeRange(range.location, range.length);
            // 得到属于高亮的 range
                    if (NSIntersectionRange(hightlightRange, lineRange).length > 0) {
```

接下来获取具体的 CGRect，注意在获取 CGRect 时还需将坐标翻转：

```objc
CGAffineTransform transform = CGAffineTransformMakeTranslation(0, contentHeight);
transform = CGAffineTransformScale(transform, 1.f, -1.f);
CGRect flipRect = CGRectApplyAffineTransform(runRect, transform);
 
// 保存链接的CGRect
NSRange nRange = NSMakeRange(range.location, range.length);
self.framesDict[NSStringFromRange(nRange)] = [NSValue valueWithCGRect:flipRect];
```

到这已经基本获取到高亮文本的位置，为什么说是基本呢？因为漏了个链接换行的问题，当链接换行显示时，就会产生多个 CTRun 对象，这些 CTRun 对应的 CGRect 都会存在 framesDict 中，当用户点击换行的链接某部分（range）时，它只能响应到 framesDict 中的一个 CGRect，而正确的做法是应该响应某个链接在 framesDict 中的所有 CGRect，只有这样才能完整的高亮出一条链接的所有部分，本质就是要将来自同一条链接的若干 CGRect 关联起来。

说了这么多，实现起来却不困难，这里采用了链接的 range 做为 key，CGRect 的数组做为 value，然后判断用户的 range 在不在链接的 range 中，若属于某条链接的 range，通过链接的 range 取出 CGRect 的数组渲染即可。

#### 3.字符串截断的处理
当 UILabel 显示不全字符串的时候，系统会在文本的最后添加“…”。同样，AttributedLabel 也提供了添加“…”的默认处理，并在此基础上提供了让用户自定义截断内容的功能。这里的实现并不难，直接截取最后一行的文本，再不断倒序删除最后一行的字符直到最后一行能容纳得下 TruncationText 为止。

首先我们还是要调用 CoreText 的 API 获取到最后一行的 range：

```objc
CTFrameRef frame = CTFramesetterCreateFrame(framesetter, CFRangeMake(0, [self length]), path, NULL);
        
CFArrayRef lines = CTFrameGetLines(frame);
NSInteger numberOfLines = CFArrayGetCount(lines);
...
NSInteger lastLineIndex = numberOfLines - 1 < 0 ? 0 : numberOfLines - 1;
CTLineRef line = CFArrayGetValueAtIndex(lines, lastLineIndex);
CFRange lastLineRange = CTLineGetStringRange(line);
```

接着使用最后一行的 range 从 AttributedString 中获取到子文本：

```objc
//截到最后一行
NSUInteger truncationAttributePosition = lastLineRange.location + lastLineRange.length;
NSMutableAttributedString *cutAttributedString = [[self attributedSubstringFromRange:NSMakeRange(0, truncationAttributePosition)] mutableCopy];
            
NSMutableAttributedString *lastLineAttributeString = [[cutAttributedString attributedSubstringFromRange:NSMakeRange(lastLineRange.location, lastLineRange.length)] mutableCopy];
```

递归调用每次删除子文本最后一个字符的方法：

```objc
- (NSMutableAttributedString *)handleLastLineAttributeString:(NSMutableAttributedString *)attributeString withTruncationText:(NSMutableAttributedString *)truncationText width:(CGFloat)width {
    CTLineRef truncationToken = CTLineCreateWithAttributedString((CFAttributedStringRef)attributeString);
    CGFloat lastLineWidth = (CGFloat)CTLineGetTypographicBounds(truncationToken, nil, nil,nil);
    CFRelease(truncationToken);
    
    if (lastLineWidth > width) {
        NSString *lastLineString = attributeString.string;
        
        NSRange r = [lastLineString rangeOfComposedCharacterSequencesForRange:NSMakeRange(lastLineString.length - truncationText.string.length - 1, 1)];
        
        [attributeString deleteCharactersInRange:r];
        
       return [self handleLastLineAttributeString:attributeString withTruncationText:truncationText width:width];
    } else {
        return attributeString;
    }
}
```

之所以递归删除是因为试过一下子截取 truncationText 的长度时会有用 `CTLineGetTypographicBounds` 计算宽度不准确的问题，不清楚这是否与不同字符的高矮胖瘦有关，如果你有更好的方法，欢迎 pr !!!

#### 4.为字符串添加附件
我最初是想用“…查看更多”截断文本，再剔除“…”后，仅把“查看更多”当作可支持高亮点击的文本，然而在实现过程中大大破坏了下边两个方法的通用性，甚至实现的效果还差强人意。

```objc
- (void)zp_highlightColor:(UIColor *)highlightColor backgroundColor:(UIColor *)backgroundColor highlightRange:(NSRange)highlightRange tapAction:(ZPTapHightlightBlock)tapAction;
 
- (NSMutableAttributedString *)zp_joinWithTruncationText:(NSMutableAttributedString *)truncationText textRect:(CGRect)textRect maximumNumberOfRows:(NSInteger)maximumNumberOfRows;
```

通常实现某个功能感到别扭时，往往都是方法没用对。最终通过查询文档及资料发现 Core Text 竟还有个 CTRunDelegate 的对象，CTRunDelegate 是 CTRun 的 delegate，它可被用来修改布局时的字形信息（glyph metrics）， 比如控制字符的 ascent、descent、width 等。换句话说，我们可以“撑开”一个字符到我们想要的高宽，在这个占位字符之上就可以添加自定义的视图（比如 UIButton）。unicode 中恰好有空白字符 `\uFFFC` 的表示，我们在字符串适当的位置插入空白字符来占位，再获取到空白字符的 CGRect 信息，就可以添加子视图在这之上了。

```objc
static void zp_deallocCallback(void *ref) {
    ZPTextRunDelegate *delegate = (__bridge_transfer ZPTextRunDelegate *)(ref);
    delegate = nil;
}
 
static CGFloat zp_ascentCallback(void *ref) {
    ZPTextRunDelegate *delegate = (__bridge ZPTextRunDelegate *)(ref);
    return delegate.ascent;
}
 
static CGFloat zp_descentCallback(void *ref) {
    ZPTextRunDelegate *delegate = (__bridge ZPTextRunDelegate *)(ref);
    return delegate.descent;
}
 
static CGFloat zp_widthCallback(void *ref) {
    ZPTextRunDelegate *delegate = (__bridge ZPTextRunDelegate *)(ref);
    return delegate.width;
}
...
CTRunDelegateCallbacks callbacks;
callbacks.version = kCTRunDelegateCurrentVersion;
callbacks.dealloc = zp_deallocCallback;
callbacks.getAscent = zp_ascentCallback;
callbacks.getDescent = zp_descentCallback;
callbacks.getWidth = zp_widthCallback;
```

最后要注意的是 CTRunDelegate 需要实现代理的委托，在委托方法中，对象并不遵循 ARC 内存管理，这里封装了 ZPTextRunDelegate 来管理属性，使用 ` __bridge_transfer`  进行内存的转换，避免了内存泄露和过早释放的 bug。获取附件的位置和高亮那块的处理类似，就不再赘述。

### 总结
本文记录了如何造一个 AttributedLabel 的轮子，相信读者结合代码一起看会发现实现简单的 Core Text 排版功能并不难，而笔者在剥离业务代码、实现通用性、封装工具类上还是遇到不少技术挑战。建议大家在平常开发中能多造点轮子锻炼锻炼技术，也能提高 iOS 技术社区的活力。同时希望大家在用惯了业界标准的 YYText 时，顺带了解下 Core Text 的使用流程。

Github 地址：[https://github.com/hawk0620/PYQFeedDemo][2]

[1]:	https://github.com/hawk0620/PYQFeedDemo
[2]:	https://github.com/hawk0620/PYQFeedDemo

[image-6]:	https://user-images.githubusercontent.com/5633917/33276265-09758578-d3d0-11e7-9690-db644433aeec.png

