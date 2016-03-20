title: iOS中富文本的AttributeNames
date: 2016-03-15 11:14:28
tags: [iOS,CoreText,NSAttributedString]
author: Kael
---
## 目录:
[1. NSFontAttributeName;](#jump1)
[2. NSParagraphStyleAttributeName;](#jump2)(这周作业内容)
[3. NSForegroundColorAttributeName;](#jump3)
[4. NSBackgroundColorAttributeName;](#jump4)
[5. NSLigatureAttributeName;](#jump5)
[6. NSKernAttributeName;](#jump6)
[7. NSStrikethroughStyleAttributeName;](#jump7)
[8. NSUnderlineStyleAttributeName;](#jump8)
[9. NSStrokeColorAttributeName;](#jump9)
[10.NSStrokeWidthAttributeName;](#jump10)
[11.NSShadowAttributeName;](#jump11)
[12.NSTextEffectAttributeName;](#jump12)
[13.NSAttachmentAttributeName;](#jump13)
[14.NSLinkAttributeName;](#jump14)
[15.NSBaselineOffsetAttributeName;](#jump15)
[16.NSUnderlineColorAttributeName;](#jump16)
[17.NSStrikethroughColorAttributeName;](#jump17)
[18.NSObliquenessAttributeName;](#jump18)
[19.NSExpansionAttributeName;](#jump19)
[20.NSWritingDirectionAttributeName;](#jump20)
[21.NSVerticalGlyphFormAttributeName;](#jump21)

___
### 1.<span id="jump1">*`NSFontAttributeName`*<span/>//字体 UIFont 
### 3.<span id="jump3">*`NSForegroundColorAttributeName`* <span/>//字颜色  UIColor
### 4.<span id="jump4">*`NSBackgroundColorAttributeName`* <span/>//背景色  UIColor

<!-- more -->

---
### 5.<span id="jump5">*`NSLigatureAttributeName`*<span/> //连写,iOS只支持@(0)和@(1)
><img src="https://img.alicdn.com/imgextra/i3/373400920/TB2D1MflFXXXXaOXXXXXXXXXXXX_!!373400920.png" width = "272" height = "201" style="margin: 0">

---
### 6.<span id="jump6">*`NSKernAttributeName`*</span>//字间距
><img src="https://img.alicdn.com/imgextra/i3/373400920/TB2CDEqlFXXXXXfXXXXXXXXXXXX_!!373400920.png" width = "373" height = "220" style="margin: 0">

---
### 7.<span id="jump7">*`NSStrikethroughStyleAttributeName`*</span> // 删除线
```objc
@(NSUnderlineStyle)枚举值
NSUnderlineStyleNone                                    = 0x00,
NSUnderlineStyleSingle                                  = 0x01,
NSUnderlineStyleThick NS_ENUM_AVAILABLE(10_0, 7_0)      = 0x02,  2~8 取值越大,线越粗
NSUnderlineStyleDouble NS_ENUM_AVAILABLE(10_0, 7_0)     = 0x09 
```
><img src="https://img.alicdn.com/imgextra/i4/373400920/TB2vm28lFXXXXc1XXXXXXXXXXXX_!!373400920.png" width = "369" height = "268" style="margin: 0">

剩下的几个枚举需要配合上面的枚举来使用
```objc
NSUnderlinePatternSolid NS_ENUM_AVAILABLE(10_0, 7_0)      = 0x0000,//实线
NSUnderlinePatternDot NS_ENUM_AVAILABLE(10_0, 7_0)        = 0x0100,//短 循环
NSUnderlinePatternDash NS_ENUM_AVAILABLE(10_0, 7_0)       = 0x0200,//长 循环
NSUnderlinePatternDashDot NS_ENUM_AVAILABLE(10_0, 7_0)    = 0x0300,//长短 循环
NSUnderlinePatternDashDotDot NS_ENUM_AVAILABLE(10_0, 7_0) = 0x0400,//长短短 循环

NSUnderlineByWord NS_ENUM_AVAILABLE(10_0, 7_0)            = 0x8000//按单词分割
```
><img src="https://img.alicdn.com/imgextra/i2/373400920/TB2KBvUlFXXXXcqXpXXXXXXXXXX_!!373400920.png" width = "371" height = "411" style="margin: 0"> 
---

### 8.<span id="jump8">*`NSUnderlineStyleAttributeName`*</span>// 下划线(值也是枚举NSUnderlineStyle的数字类型-@(NSUnderlineStyle)参考NSStrikethroughStyleAttributeName)
><img src="https://img.alicdn.com/imgextra/i4/373400920/TB2qJEblFXXXXXhXpXXXXXXXXXX_!!373400920.png" width = "371" height = "372" style="margin: 0"> 

---
### 9.<span id="jump9">*`NSStrokeColorAttributeName`*</span>// 笔画宽度和当前字的pointSize(字体大小)的比例,
正数真空效果
><img src="https://img.alicdn.com/imgextra/i4/373400920/TB2d9UylFXXXXX4XXXXXXXXXXXX_!!373400920.png" width = "372" height = "235" style="margin: 0">

---

### 10.<span id="jump10">*`NSStrokeColorAttributeName`*</span>//NSStrokeColorAttributeName的颜色
><img src="https://img.alicdn.com/imgextra/i1/373400920/TB20GkzlFXXXXafXXXXXXXXXXXX_!!373400920.png" width = "373" height = "121" style="margin: 0">

---
### 11.<span id="jump11">*`NSShadowAttributeName`*</span> //阴影,参考NSShadow
><img src="https://img.alicdn.com/imgextra/i1/373400920/TB2Wmv1lFXXXXbRXpXXXXXXXXXX_!!373400920.png" width = "371" height = "72" style="margin: 0">

---
### 12.<span id="jump12">*`NSTextEffectAttributeName`*</span>//凸版印刷体(现在就只有NSTextEffectLetterpressStyle一个值)
凸版印刷替效果是给文字加上奇妙阴影和高光，让文字看起有凹凸感，像是被压在屏幕上(这个描述真是有够夸张 = =!)
><img src="https://img.alicdn.com/imgextra/i3/373400920/TB29UgalFXXXXaXXpXXXXXXXXXX_!!373400920.png" width = "466" height = "47" style="margin: 0">


---
### 13.<span id="jump13">*`NSAttachmentAttributeName`*</span>//图文混排相关
```objc
NSTextAttachment *attach=[[NSTextAttachment alloc]init];
    attach.image=[UIImage imageNamed:@"1178298162bf1917"];
    [base insertAttributedString:[NSAttributedString attributedStringWithAttachment:attach] atIndex:6];
```
><img src="https://img.alicdn.com/imgextra/i3/373400920/TB2v12_lFXXXXa3XpXXXXXXXXXX_!!373400920.png" width = "369" height = "177" style="margin: 0">

---
### 14.<span id="jump14">*`NSLinkAttributeName`*</span>//链接,但是不负责点击的处理
```objc
NSMutableAttributedString *base=[[NSMutableAttributedString alloc]initWithString:string attributes:nil];
NSRange rang=NSMakeRange(0, base.length);
[base addAttribute:NSLinkAttributeName value:[NSURL URLWithString:@"http://www.google.com"] 
range:[string rangeOfString:@"http://www.google.com"]];
```
><img src="https://img.alicdn.com/imgextra/i3/373400920/TB2ud.vlFXXXXbzXXXXXXXXXXXX_!!373400920.png" width = "365" height = "45" style="margin: 0">

一般配合UITextView使用
```objc
- (BOOL)textView:(UITextView *)textView shouldInteractWithURL:(NSURL *)URL inRange:(NSRange)characterRange
{
    NSLog(@"=============%@",URL);
    return YES;
}
```

---
### 15.<span id="jump15">*`NSBaselineOffsetAttributeName`*</span>//离BaseLine的距离
[什么是BaseLine?](https://developer.apple.com/library/mac/documentation/TextFonts/Conceptual/CocoaTextArchitecture/FontHandling/FontHandling.html#//apple_ref/doc/uid/TP40009459-CH5-SW1)
![](https://developer.apple.com/library/mac/documentation/TextFonts/Conceptual/CocoaTextArchitecture/Art/glyph_metrics_2x.png)
><img src="https://img.alicdn.com/imgextra/i4/373400920/TB29L.tlFXXXXcsXXXXXXXXXXXX_!!373400920.png" width = "367" height = "75" style="margin: 0">

---
### 16.<span id="jump16">*`NSUnderlineColorAttributeName`*</span>//下划线的颜色
相关属性(NSUnderlineStyleAttributeName)

### 17.<span id="jump17">*`NSStrikethroughColorAttributeName`*</span>//中划线的颜色
相关属性(NSStrikethroughStyleAttributeName)

---
### 18.<span id="jump18">*`NSObliquenessAttributeName`*</span>//倾斜
><img src="https://img.alicdn.com/imgextra/i3/373400920/TB2le.DlFXXXXaDXXXXXXXXXXXX_!!373400920.png" width = "371" height = "383" style="margin: 0">

---
### 19.<span id="jump19">*`NSExpansionAttributeName`*</span>//"胖"or "瘦"(拉伸or压缩)
><img src="https://img.alicdn.com/imgextra/i3/373400920/TB2ru.qlFXXXXc_XXXXXXXXXXXX_!!373400920.png" width = "370" height = "286" style="margin: 0">

---
### 20.<span id="jump20">*`NSWritingDirectionAttributeName`*</span>//文字的排布顺序(从左到右还是从右到左)
><img src="https://img.alicdn.com/imgextra/i3/373400920/TB2YfgLlFXXXXX8XXXXXXXXXXXX_!!373400920.png" width = "686" height = "273" style="margin: 0">

```objc
//不太明白这个枚举值的两个意思...之后如果有机会明白再解释他们
typedef NS_ENUM(NSInteger, NSWritingDirectionFormatType) {
    NSWritingDirectionEmbedding     = (0 << 1),
    NSWritingDirectionOverride      = (1 << 1)
} NS_ENUM_AVAILABLE(10_11, 9_0);
```


---
### 21.<span id="jump21">*`NSVerticalGlyphFormAttributeName`*</span>//0横,1竖
在iOS上只有横排,不过可以通过CoreText来修改成竖排
