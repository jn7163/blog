---
title: 微信小程序学习手记（一）：自定义可滚动TabBar
date: 2018-07-26 21:20:27
tags:
  - mini program
  - wechat
---

暑假尝试开发了一下微信小程序，发现整体难度并不会很大。框架基本原理就是 MVVM，接触过 Vue.js 等类似框架的人上手相当容易。不过微信生态圈还是给人相对封闭的感觉，自创的 wxml + wxsss + wxs 相比 html + css + js 显得很不健全。当然这不是重点，今天的重点是谈谈自定义 TabBar 。

<!--more-->

有人说微信已经自带 TabBar 了，会什么要重复发明轮子呢？原因就在于微信的官方组件将 tab 的数量限制在 2 ~ 5，并且位置只能置顶或沉底，在一些特定场景可能无法满足需求。所以我开始尝试自定义 TabBar。

## 问题一：界面布局

写前端自然要考虑布局，而小程序布局与传统网页布局方式就有所不同。传统网页基于盒模型（Box Model），广泛使用 position + display + float 这样的组合；而微信小程序采用了较新的 Flex 布局。不了解的可以参考这篇文章：[Flex 布局语法教程](http://www.runoob.com/w3cnote/flex-grammar.html)。

然后开始实现 TabBar，考虑以下两个问题：

- 如何解决开放数量限制可能导致的内容溢出？
- 如何让 TabBar 固定在页面的任意位置？

第一个问题通过微信组件 [scroll-view](https://developers.weixin.qq.com/miniprogram/dev/component/scroll-view.html) 可以解决，让 TabBar 横向（或竖向）滚动起来就可以了。第二个问题可以通过 position 属性解决，当然也可以跟随 Flex 布局自然流动。贴上源码：

```html
<view class="container">
  <scroll-view class="tab-bar" scroll-x="true">
    <view class="tab-item" wx:for="{{tabs}}">
      {{item}}
    </view>
  </scroll-view>
</view>
```
```css
.tab-bar {
  /* Box Model */
  display: flex;
  flex-direction: row;
  white-space: nowrap;
  width: 100%;
  /* Position */
  position: absolute;
  top: 0;
}

.tab-bar .tab-item {
  /* Box Model */
  display: inline-block;
  width: 25%;
}
```

这里特别注意一下 white-space 属性，防止溢出元素换行；以及 display: inline-block属性，它让 tab-item 以行内元素而非块级元素显示，同时 block 的特性使得我们可以让文字水平居中。

## 问题二：Tab 样式

处理 Tab 样式时碰上两个小问题：

- 文字垂直居中
- 当前 Tab 高亮显示

行内元素的垂直居中相信有学过 CSS 的都能解决，将 height 与 line-height 设为同一数值即可。在高亮显示时，我希望给当前元素添加边框 —— 但众所周知，对行内元素添加边框是非法的；于是最终参考[这篇博客](https://blog.csdn.net/sophie_u/article/details/71745125)采用 :after 伪元素的方案解决了。依旧贴上源码：

```css
.tab-bar {
  box-shadow: 5rpx 5rpx 5rpx #ccc;
  height: 100rpx;
}

.tab-bar .tab-item {
  line-height: 100rpx;
  text-align: center;
}

.tab-bar .tab-item.active {
  border-radius: 10%;
  background-color: #eee;
  color: skyblue;
  position: relative;
}

.tab-bar .tab-item.active:after {
  background-color: skyblue;
  content: "";
  display: block;
  height: 5rpx;
  width: 100%;
  /* position */
  position: absolute;
  bottom: 0;
}
```

## 问题三：js 逻辑处理

最后当然还需要让 Tab 响应点击事件来实现 Tab 之间的切换，这个问题可以通过绑定一个 current 变量轻松解决。并不困难就直接上源码了：

```html
<view class="tab-item {{current===index?'active':''}}" wx:for="{{tabs}}" data-index="{{index}}" bindtap="switchTab">
  {{item}}
</view>
```
```javascript
const app = getApp();

Page({
  data: {
    current: 0,
    tabs: ['Windows', 'Ubuntu', 'CentOS', 'Manjaro', 'MacOS']
  },
  switchTab: function(e) {
    this.setData({
      current: e.target.dataset.index
    });
  }
});
```

## 成果

如下图：

[![img](https://i.loli.net/2018/09/16/5b9e5978e0c6e.gif)](https://i.loli.net/2018/09/16/5b9e5978e0c6e.gif)