---
layout: blog
title: 如何判断 TextView 内容是否被缩略
date: 2016-03-07 10:43:46
categories: blog
tags: [Android, TextView, 缩略, Ellipse]
description: 奇技淫巧之如何判断一个 TextView 的内容是否被省略了
author: chen
---

## What
TextView 中所谓的缩略其实如下, 实际上是在显示文本时将最后一个字符替换成 … 而已:
{% asset_img ellipse_example.png 所谓的缩略 %}

## Why
产品需求而已, 不细说了, 都是泪啊

## How

### 机制
TextView 中的文本到底是如何排版以及显示的呢? 答案是 [Layout][1]

在我们调用 `textView.setText("Some very long text")` 后, 真正展示在界面上的可能是 "Some very l…", 然而此时我们调用 `textView.getText()` 得到的却依旧是 "Some very long text".
{%asset_img text_get_text.png textView.getText() 的结果 %}

因吹丝停!

那么这时我们通过 `textView.getLayout().getText()` 拿到的是什么呢? 答案是 "Some very l…ng text".
{%asset_img text_layout_get_text.png textView.getLayout.getText()的结果 %}

WTH?!

原来只是把**字符替换**了一下而已

### 实践 
了解到 TextView 的 trick 之后, 我们也可以利用它来判断当前显示的文本是否被缩略了:
```Java
/**
 * 判断一个 TextView 显示的内容是否被缩略了
 */
public static boolean isTextEllipse(@NonNull TextView textView) {
	try {
		CharSequence rawText = textView.getText();
		CharSequence displayText = textView.getLayout().getText();
		return !TextUtils.equals(rawText, displayText);
	} catch(Exception ignored) {
		// getLayout() 可能返回 null
		return false;
	}
}
```

## More
由于渲染机制, 文中的方法需要在 textView.setText 之后调用, 并且需要 post 到主线程中才能生效:
```Java
textView.setText("Some very long text");
textView.post(new Runnable() {
	public void run() {
		if (isTextEllipse(textView) {
			// do some work
		} else {
			// some other work
		}
	}
});
```

[1]:(http://developer.android.com/intl/es/reference/android/text/Layout.html) 
