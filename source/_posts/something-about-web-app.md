---
title: 关于移动端开发中遇到的坑
date: 2017-07-30 22:54:06
tags: 
- 移动端web开发
- JavaScript
categories: 前端开发
toc: true
---
# 前言

三月中旬跳完槽之后就好好久没写博客了，跳到某公司之后，怀揣着满腔热水的我又投入了工作中，从面试、办理离职到入职只用了一个星期。这效率也没谁了，入职之后给了一个小项目，用Vue全家桶开发一个简易的个人博客。因为之前自学了解过，于是很快就上手，原本要求用两周的时间用了三天就完成了全部的功能，于是就一周之后就开始安排到项目组进行实际的开发中了，最后被安排到招聘组负责校招的前端开发。一去就搞事情，给我安排了后台移动端的开发，就是为了方便领导手机上使用，基于内部项目都是用了Vue，最后确定用Vue+一个移动端的基于Vue的UI框架Vux进行开发，于是就进入了移动端的踩坑之旅，之前比较少接触移动端开发。
项目开发是基于Vue的UI框架Vux，其实就是一套基于We-UI的一套移动端UI框架，但根据实际情况，一些布局还是得自己去重构。于是就愉快的踩起了移动端开发的坑。
<!-- more -->

# 滚动穿透问题

滚动穿透是指在移动端当有 fixed 遮罩背景和弹出层时，在屏幕上滑动能够滑动背景下面的内容。网上整理了解决方案，但有些还是存在一定的问题：

## 设置overflow为hidden

```css
.modal-open {
    &, body {
        overflow: hidden;
        height: 100%
    }
}
```

即当弹出层弹出时在html上添加.modal-open,禁用 html 和 body 的滚动条,但实际用上就会发现：

1. 由于 html 和 body的滚动条都被禁用，弹出层后页面的滚动位置会丢失，需要用 js 来计算原来滚动的位置，在弹出时保持滚动位置；
2. 杯具的是页面的背景还是能够有滚的动的效果

## js 之 touchmove + preventDefault

```js
modal.addEventListener('touchmove', function(e) {
  e.preventDefault();
}, false);
```

即通过阻止移动端touchmove事件，但实际用上会发现弹出层需要滚动时也会被阻止

## 最后解决方案：position: fixed

```css
body.modal-open {
    position: fixed;
    width: 100%;
}
```

这种方式同样当弹出层弹出时滚动条会丢失，所以还需要使用js来保存滚动条的位置，在关闭弹出层时将滚动位置还原；

```js
var ModalHelper = (function(bodyCls) {
  var scrollTop; // 在闭包中定义一个用来保存滚动位置的变量
  return {
    afterOpen: function() { //弹出之后记录保存滚动位置，并且给body添加.modal-open
      scrollTop = document.scrollingElement.scrollTop;
      document.body.classList.add(bodyCls);
      document.body.style.top = -scrollTop + 'px';
    },
    beforeClose: function() { //关闭时将.modal-open移除并还原之前保存滚动位置
      document.body.classList.remove(bodyCls);
      document.scrollingElement.scrollTop = scrollTop;
    }
  };
})('modal-open');
```

本人亲测确实比较完美解决了移动端滚动问题😊

# 移动端输入框被键盘挡住问题

由于所开发的页面内嵌在公司的一个APP中，有一个类似聊天窗口的界面，测试的时候发现在部分安卓机中输入框被完全遮挡住，踩这个坑时在网上找了好多资料，好像都没有一套完整的解决办法，先看其中一种解决办法，可以解决绝大数安卓机上面的问题：

```js
if(/Android/.test(navigator.appVersion)) {
   window.addEventListener("resize", function() {
     if(document.activeElement.tagName=="INPUT" || document.activeElement.tagName=="TEXTAREA") {
       document.activeElement.scrollIntoView();
     }
  })
}
```

即在安卓机中通过监听当窗口resize时，判断当前获得焦点的元素是否为输入框，再调用该元素的scrollIntoView()，即将该元素展示在当前窗口的可视区域。由于只有scrollIntoView被各浏览器均支持，所以这个方法最为常用。
使用这段代码之后，在微信或者其他浏览器测试时有效果，但因为是需要内嵌在自家APP上，使用这段代码一直没有解决输入框被挡住的问题，最后测试才发现，APP内置浏览器在聚焦输入框弹出键盘根本没有触发窗口的resize事件，瞬间心中万马奔腾(＞﹏＜)~~~，后面在借鉴了某阿里的一个网页版的聊天界面，发现它是通过获取输入框焦点将输入框定位到窗口略高于输入框的位置，在失去焦点键盘弹回时再恢复到底部，于是通过这种方式处理，暂时比较暴力的解决了在安卓上该APP上输入框被挡住的问题，这种方法显然是不完美的，比如由于无法监听resize事件，而且使用的键盘高度不固定，所以只能大概的将高度设置保持在屏幕一半偏上一点。保证绝大数情况下输入框在键盘之上显示。

# IOS滚动不平滑的问题

在移动端特别是iOS中，当滚动屏幕时会发现手指一拿开滚动就停止，这种用户体验效果很不好，有种给用户滚动卡顿的感觉。CSS3中的-webkit-overflow-scrolling属性可以完美的解决这个问题，该属性可控制元素在移动设备上是否使用滚动回弹效果:

1. auto
> 使用普通滚动, 当手指从触摸屏上移开，滚动会立即停止。

2. touch
> 使用具有回弹效果的滚动, 当手指从触摸屏上移开，内容会继续保持一段时间的滚动效果。继续滚动的速度和持续的时间和滚动手势的强烈程度成正比。同时也会创建一个新的堆栈上下文。
