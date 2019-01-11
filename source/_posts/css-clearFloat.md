---
title: CSS之清除浮动
date: 2017-02-16 14:19:03
tags: CSS
categories: 前端开发
toc: true
---

面试的时候，关于清除浮动的问题经常会被考察，一开始对清除浮动了解不深，只知道css有个clear属性。在平时开发中也经常会用到float布局方式，之前因为了解不深，所以使用这种布局的时候经常出现问题。后面查找资料才发现原来清除浮动后面大有学问，了解清除浮动有利于我们对布局的开发，下面是对了解之后作出的一些总结，可能总结的不全面。  

在了解清除浮动之前，先看下什么是BFC。

<!-- more -->

# 了解BFC

首先看看W3C对BFC是怎么定义的：
>Floats, absolutely positioned elements, block containers (such as inline-blocks, table-cells, and table-captions) that are not block boxes, and block boxes with ‘overflow’ other than ‘visible’ (except when that value has been propagated to the viewport) establish new block formatting contexts for their contents.  
In a block formatting context, boxes are laid out one after the other, vertically, beginning at the top of a containing block. The vertical distance between two sibling boxes is determined by the ‘margin’ properties. Vertical margins between adjacent block-level boxes in a block formatting context collapse.  
In a block formatting context, each box’s left outer edge touches the left edge of the containing block (for right-to-left formatting, right edges touch). This is true even in the presence of floats (although a box’s line boxes may shrink due to the floats), unless the box establishes a new block formatting context (in which case the box itself may become narrower due to the floats).

翻译过来就是：
>浮动元素和绝对定位元素，非块级盒子的块级容器（例如 inline-blocks, table-cells, 和 table-captions），以及overflow值不为“visiable”的块级盒子，都会为他们的内容创建新的块级格式化上下文。  
在一个块级格式化上下文里，盒子从包含块的顶端开始垂直地一个接一个地排列，两个盒子之间的垂直的间隙是由他们的margin 值所决定的。两个相邻的块级盒子的垂直外边距会发生叠加。  
在块级格式化上下文中，每一个盒子的左外边缘（margin-left）会触碰到容器的左边缘(border-left)（对于从右到左的格式来说，则触碰到右边缘），即使存在浮动也是如此，除非这个盒子创建一个新的块级格式化上下文。

BFC就是“块级格式化上下文”的意思，创建了BFC的元素就是一个独立的盒子，不过只有Block-level box可以参与创建BFC，它规定了内部的Block-level Box如何布局，并且与这个独立盒子里的布局不受外部影响，当然它也不会影响到外面的元素。

## BFC的特性

* 内部的Box会在垂直方向，从顶部开始一个接一个地放置。
* Box垂直方向的距离由margin决定。属于同一个BFC的两个相邻Box的margin会发生叠加
* 每个元素的margin box的左边， 与包含块border box的左边相接触(对于从左往右的格式化，否则相反)。即使存在浮动也是如此。
* BFC的区域不会与float box叠加。
* BFC就是页面上的一个隔离的独立容器，容器里面的子元素不会影响到外面的元素，反之亦然。
* 计算BFC的高度时，浮动元素也参与计算。

## 触发BFC的属性

* float 除了none以外的值
* overflow 除了visible 以外的值（hidden，auto，scroll ）
* display (table-cell，table-caption，inline-block, flex, inline-flex)
* position值为（absolute，fixed）

# 清除浮动

在非IE浏览器（如Firefox）下，当容器的高度为auto，且容器的内容中有浮动（float为left或right）的元素，在这种情况下，容器的高度不能自动伸长以适应内容的高度，使得内容溢出到容器外面而影响（甚至破坏）布局的现象。这个现象叫浮动溢出，为了防止这个现象的出现而进行的CSS处理，就叫CSS清除浮动。

> 清除浮动的本质就是让元素标签闭合，即触发其BFC,所以以上可以触发BFC的可以清除浮动,此外css还有一个clear属性也可以清除浮动

## 方法

方法一：使用额外标签法，即使用带clear属性的空元素  

优点：简单，代码少，浏览器兼容性好。

缺点：需要添加大量无语义的html元素，代码不够优雅，后期不容易维护。

``` html
.box {
  background-color: gray;
  border: solid 1px black;
  }

.box img {
  float: left;
  }

.box p {
  float: right;
  }

.clear {
  clear: both;
  }

<div class="box">
    <img src="news-pic.jpg" />
    <p>some text</p>
    <div class="clear"></div>
</div>
``` 

方法二：利用CSS的overflow属性

给浮动元素的容器添加overflow:hidden;或overflow:auto;可以清除浮动，另外在 IE6 中还需要触发 hasLayout ，例如为父元素设置容器宽高或设置 zoom:1。明显的缺点是当容器内部的内容过多时会被隐藏

``` html
.box {
  background-color: gray;
  border: solid 1px black;
  overflow: hidden;
  *zoom: 1;
  }

.box img {
  float: left;
  }

.box p {
  float: right;
  }

<div class="box">
    <img src="news-pic.jpg" />
    <p>some text</p>
</div>
``` 

方法三：给浮动的元素的容器添加浮动

给浮动元素的容器也添加上浮动属性即可清除内部浮动，但是这样会使其整体浮动，影响布局，不推荐使用。

方法四：给父元素定高

父容器因为子元素浮动造成高度撑不开，可以给父元素定高度，缺点明显是当父容器高度固定了，不够灵活。

方法五：使用css的:after伪元素（现在项目中清除浮动就是采用这种方法，兼容性跟友好性都比较好）

after伪元素和zoom：1结合可以解决当前绝大多数浏览器浮动问题，zoom:1可以触发IE的hasLayout，具体实现代码如下：

``` html
.box {
  background-color: gray;
  border: solid 1px black;
  overflow: hidden;
  *zoom: 1;
  }

.box img {
  float: left;
  }

.box p {
  float: right;
  }

.clearfix:after{
  content: "."; 
  display: block; 
  height: 0; 
  clear: both; 
  visibility: hidden;  
  }

.clearfix {
  /* 触发 hasLayout */ 
  zoom: 1;
  }
<div class="box clearfix">
    <img src="news-pic.jpg" />
    <p>some text</p>
</div>
```

## 总结

通过上面的例子，清除浮动的方法可以分成两类：
一是利用 clear 属性，包括在浮动元素末尾添加一个带有 clear: both 属性的空 div 来闭合元素，其实利用 :after 伪元素的方法也是在元素末尾添加一个内容为一个点并带有 clear: both 属性的元素实现的。  
二是触发浮动元素父元素的 BFC (Block Formatting Contexts, 块级格式化上下文)，使到该父元素可以包含浮动元素。