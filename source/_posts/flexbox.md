---
title: flexbox学习小记
date: 2017-02-23 17:21:56
tags: CSS
toc: true
---
因为现在的公司要求布局的兼容性要兼容到至少IE7，所以对于flexbox这种布局也没有了解太多，之前有听说过flex布局，听起来是一种比较高大上，可以自适应不同尺寸屏幕的布局方式，但是没有正式的去学习了解。flexbox是CSS3算是比较新的一个特性，最近刚好有时间也有兴趣就了解了，觉得还是要做一下总结，不然很快又忘了😰。  

flexbox布局意为弹性布局，是css3的弹性盒子模式，用它可以告别float布局，也可以简单实现垂直居中，并且具有响应式，就是可以自动调整，计算元素在容器空间中的大小，可以说：完美~~~（此处应有标志性动作)  

> Flex布局背后的主要思想是给容器控制项目（Flex项目）的宽度、高度的能力，使用Flex项目可以自动填满容器的可用空间（主要是适应所有类型的显示设备和屏幕大小）。Flex容器使用Flex项目可以自动放大与收缩，用来填补可用的空闲空间。

<!-- more -->

先看下flexbox布局对各大浏览器的支持情况，基本也就告别低版本的浏览器了：

<div align=center>
  <img src="https://github.com/huangzhuangjia/BlogImages/blob/master/img/img04.jpeg?raw=true" alt="兼容性示意图">
</div>

下面我们来简单了解一下flex布局的基本概念：

# flex项目排列方式：

<div align=center>
 <img src="http://www.w3cplus.com/sites/default/files/blogs/2015/1504/flexbox.png" alt="示意图">
</div>

* main axis:Flex容器的主轴主要用来配置Flex项目。注意，它不一定是水平，这主要取决于flex-direction属性。
* main-start | main-end:Flex项目的配置从容器的主轴起点边开始，往主轴终点边结束。
* main size:Flex项目的在主轴方向的宽度或高度就是项目的主轴长度，Flex项目的主轴长度属性是width或height属性，由哪一个对着主轴方向决定。
* cross axis:与主轴垂直的轴称作侧轴，是侧轴方向的延伸。
* cross-start | cross-end:伸缩行的配置从容器的侧轴起点边开始，往侧轴终点边结束。
* cross size:Flex项目的在侧轴方向的宽度或高度就是项目的侧轴长度，Flex项目的侧轴长度属性是width或height属性，由哪一个对着侧轴方向决定。

# flex布局拥有的属性：

## 父容器的属性

* display:flex | inline-flex。 表明这个容器是flex布局。flex是块伸缩容器，inline-flex是内联伸缩容器。
* flex-direction: row | row-reverse | column | column-reverse; 表明容器里面的子元素的排列方向。
* flex-wrap: nowrap | wrap | wrap-reverse; 如果子元素溢出父容器的时候是否进行换行。
* justify-content: flex-start | flex-end | center | space-between | space-around; 这一个容器子元素横向排版在容器的哪个位置
* align-items: flex-start | flex-end | center | baseline | stretch; 这个容器子元素纵向排版在容器的哪个位置
* align-content: flex-start | flex-end | center | space-between | space-around | stretch; 当容器内有多行项目的时候，项目的布局

## 子元素的属性

* order: ; 子元素的排序
* flex-grow: ; 分配剩余空间的比例
* flex-shrink: ; 分配溢出空间的比例
* flex-basis: | auto;
* flex: none | [ <'flex-grow'> <'flex-shrink'>? || <'flex-basis'> ] 在容器中占比
* align-self: auto | flex-start | flex-end | center | baseline | stretch; 特定某个子元素的排布情况

# 属性详解

## Flex容器属性：

### display属性

display定义了一个Flex容器，为其内容建立新的伸缩格式化上下文。其中flex是块伸缩容器，inline-flex是内联伸缩容器。

```css
  .container{
      display: flex;/* or inline-flex*/
  }
```

> 如果元素display的值指定为inline-flex，而且元素是一个浮动元素或绝对定位元素，则display的计算值是flex。

### flex-direction属性

flex-direction定义了flex容器里面的flex项目的排列方向，水平排列和竖直排列

```css
  .container { 
    flex-direction: row | row-reverse | column | column-reverse; 
  }
```

* row(默认值): 主轴为水平方向，从左向右排列
* row-reverse: 主轴为水平方向，从右向左排列
* column: 主轴为垂直方向，从上到下排列
* column-reverse: 主轴为垂直方向，从下向上排列

### flex-wrap属性

flex-wrap定义了当一行或者一列排不上，flex项目是否换行，默认是不换行。

```css
  .container {
    flex-wrap: nowrap | wrap | wrap-reverse;
  }
```

* nowrap(默认值): 单行显示不换行

 ![示意图](https://github.com/huangzhuangjia/BlogImages/blob/master/img/img01.jpeg?raw=true)

* wrap: 多行显示换行，第一行在上方

 ![示意图](https://github.com/huangzhuangjia/BlogImages/blob/master/img/img02.jpeg?raw=true)

* wrap-reverse: 多行显示换行，第一行在下方

 ![示意图](https://github.com/huangzhuangjia/BlogImages/blob/master/img/img03.jpeg?raw=true)

### flex-flow属性

 flex-flow是flex-direction属性和flex-wrap属性的简写形式，默认值为row nowrap。道理就跟border: 1px solid red类似，多个值写在同一行。

### justify-content属性

 justify-content定义了flex项目在flex容器主轴中的对齐方式。

 ```css
  container{
    justify-content: {flex-start | flex-end | center | space-between | space-around;
  }
```

<div align=center>
  <img src="http://www.ruanyifeng.com/blogimg/asset/2015/bg2015071010.png" alt="示意图">
</div>

* flex-start（默认值）：flex项目在主轴方向上进行左对齐
* flex-end：flex项目在主轴方向上进行右对齐
* center： flex项目在主轴方向上进行居中
* space-between：flex项目两端对齐，项目之间的间隔都相等。
* space-around：每个项目两侧的间隔相等。所以，项目之间的间隔比项目与边框的间隔大一倍。

### align-items属性

align-items定义了flex项目在flex容器竖轴上的对齐方式。类似justify-content属性，只不过是方向不同。

 ```css
  container{
    align-items: {flex-start | flex-end | center | baseline | stretch;
  }
```

<div align=center>
  <img src="http://www.ruanyifeng.com/blogimg/asset/2015/bg2015071011.png" alt="示意图">
</div>

* flex-start（默认值）：flex项目在flex容器竖轴方向起点对齐。
* flex-end：flex项目在flex容器竖轴方向终点对齐。
* center：flex项目在flex容器竖轴方向中点对齐。
* baseline: 项目的第一行文字的基线对齐。
* stretch：如果项目未设置高度或设为auto，将占满整个容器的高度。

### align-content属性

align-content属性定义了多根轴线的对齐方式。当伸缩容器的侧轴还有多余空间时，align-content属性可以用来调准伸缩行在伸缩容器里的对齐方式，这与调准伸缩项目在主轴上对齐方式的justify-content属性类似。

```css
  container{
    align-items: {flex-start | flex-end | center | baseline | stretch;
  }
```

<div align=center>
  <img src="http://www.ruanyifeng.com/blogimg/asset/2015/bg2015071012.png" alt="示意图">
</div>

* flex-start：与竖轴方向的起点对齐。
* flex-end：与竖轴方向的终点对齐。
* center：与竖轴方向的中点对齐。
* space-between：与竖轴的两端对齐，轴线之间的间隔平均分布。
* space-around：每根轴线两侧的间隔都相等。所以，轴线之间的间隔比轴线与边框的间隔大一倍。
* stretch（默认值）：轴线占满整个竖轴方向。

## Flex项目属性：

### order属性

order属性定义项目的排列顺序。数值越小，排列越靠前，默认为0。

 ```css
  .item {
    order: <integer>;
  }
 ```

 > 根据order重新排序伸缩项目。有最小（负值最大）order的伸缩项目排在第一个。若有多个项目有相同的order值，这些项目照文件顺序排。这个步骤影响了伸缩项目生盒树成的盒子的顺序，也影响了后面的演算法如何处理各项目。

### flex-grow属性

 flex-grow属性定义项目的放大比例，默认为0，即如果存在剩余空间，也不放大。flex-grow取负值将失效。

 ```css
  .item {
    flex-grow: <number>; /* default 0 */
  }
 ```

 <div align=center>
  <img src="http://www.ruanyifeng.com/blogimg/asset/2015/bg2015071014.png" alt="示意图">
 </div>

 如上图所示，若所有flex项目的flex-grow属性都为1，则flex容器中的flex项目尺寸都相等，若其中一个为2其他为1，则为其他flex项目的两倍。

### flex-shrink属性

flex-shrink属性则与flex-grow属性相反，定义了项目的缩小比例，默认为1，即如果空间不足，该项目将缩小。与flex-grow一样，flex-shrink取负值将失效。

```css
  .item {
    flex-shrink: <number>; /* default 1 */
  }
```

### flex-basis属性

flex-basis属性定义了Flex项目在分配Flex容器剩余空间之前的一个默认尺寸，通俗一点是说在flex-grow和flex-shrink属性调整它的大小以适应Flex容器之前，可以指定Flex项目的初始大小。它的默认值为auto，即项目的本来大小。flex-basis可以取任何用于width属性的任何值，比如 % || em || rem || px等。

> 注意：如果flex-basis属性的值是0时，也需要使用单位。即flex-basis: 0px不能写成flex-basis:0。

```css
 .item {
   flex-basis: <length> | auto; /* default auto */ 
  }
```

### flex属性

flex属性是flex-grow, flex-shrink 和 flex-basis的简写，默认值为0 1 auto。  
> 注意：flex-grow第一，然后是flex-shrink，最后是flex-basis。缩写成GSB。

```css
 .item { 
   flex: 0 1 auto;/*flex-grow: 0; flex-shrink: 1; flex-basis: auto;*/
 }
```

flex常见值:

1. flex: 0 auto,flex: initial与flex: 0 1 auto相同。根据width／height属性决定元素的尺寸。（如果项目的主轴长度属性的计算值为auto，则会根据其内容来决定元素尺寸。）当剩余空间为正值时，伸缩项目无法伸缩，但当空间不足时，伸缩项目可收缩至其最小值。

2. flex: auto与flex: 1 1 auto相同。根据width／height属性决定元素的尺寸，但是完全可以伸缩，会吸收主轴上剩下的空间。

3. flex: none与flex: 0 0 auto相同。根据width／height属性决定元素的尺寸，但是完全不可伸缩，基本上就是一个固定宽度的元素，初始宽度是基于flex项目内容大小。注意：即使在空间不够而溢出的情况下，伸缩项目也不能收缩。

4. flex: <positive-number>与flex: 1 0px相同。该值使元素可伸缩，并将伸缩基准值设置为零，导致该项目会根据设置的比率占用伸缩容器的剩余空间。如果一个伸缩容器里的所有项目都使用此模式，则它们的尺寸会正比于指定的伸缩比率。

默认状态下，伸缩项目不会收缩至比其最小内容尺寸（最长的英文词或是固定尺寸元素的长度）更小。网页作者可以靠设置min-width或min-height属性来改变这个默认状态.

### align-self属性

align-self属性允许单个项目有与其他项目不一样的对齐方式，可覆盖align-items属性。默认值为auto，表示继承父元素的align-items属性，如果没有父元素，则等同于stretch。

```css
  .item { 
    align-self: auto | flex-start | flex-end | center | baseline | stretch; 
  }
``` 

<div align=center>
  <img src="http://www.ruanyifeng.com/blogimg/asset/2015/bg2015071016.png" alt="示意图">
</div>

* flex-start:伸缩项目在侧轴起点边的外边距紧靠住该行在侧轴起始的边。
* flex-end:伸缩项目在侧轴终点边的外边距靠住该行在侧轴终点的边 。
* center:伸缩项目的外边距盒在该行的侧轴上居中放置。（如果伸缩行的尺寸小于伸缩项目，则伸缩项目会向两个方向溢出相同的量）。
* baseline:如果伸缩项目的行内轴与侧轴为同一条，则该值和flex-start等效。其它情况下，该值将参与基线对齐。所有参与该对齐方式的伸缩项目将按下列方式排列：首先将这些伸缩项目的基线进行对齐，随后其中基线至侧轴起点边的外边距距离最长的那个项目将紧靠住该行在侧轴起点的边。
* stretch:如果侧轴长度属性的值为auto，则此值会使项目的外边距盒的尺寸在遵照min/max-width/height属性的限制下尽可能接近所在行的尺寸。

> 博客中有些地方是参考阮老师的文章，再自己作总结的，具体可以参考阮老师的[《Flex布局教程》](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html)