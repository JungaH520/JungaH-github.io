---
title: 移动端开发问题小记
date: 2019-07-03 12:25:01
tags: 
- JavaScript
- 移动端开发
categories: 前端开发
toc: true
---

## 关于ios键盘挡住问题（适用于input置底，键盘弹起窗口可滚动）

```js
 /**
   * 解决ios键盘被挡住input问题，适用于input置底，键盘弹起窗口可滚动
   * @param {*} input input类名或选择器
   */
  repairIosInputHidden(input) {
    if (this.isIos()) {
      const inputEl = typeof input === 'string' ? document.querySelector(input) : input;
      let timeout = null;
      // 记录窗口滚动前的滚动高度
      const scrollElement = document.scrollingElement || document.body || document.documentElement;
      const bfscrolltop = scrollElement.scrollTop || window.pageYOffset;
      const setScrollTop = (h) => {
        scrollElement.scrollTop = h;
      };
      const scrollToBottom = () => {
        const scrollHeight = document.documentElement.scrollHeight || document.body.scrollHeight;
        setScrollTop(scrollHeight);
      };
      // input聚焦键盘弹起，获取窗口可滚动高度scrollHeight，延时赋值，此时已经是最大可滚动高度了，
      // 即键盘弹出后滚动条会滚动到底部，input会显示在键盘之上
      inputEl.onfocus = () => {
        // 让input显示在可视区域，兼容部分机型原生键盘挡住问题
        inputEl.scrollIntoView(true);
        timeout = setTimeout(() => {
          scrollToBottom();
        }, 300);
      };
      // input失去焦点后，恢复原来滚动位置
      inputEl.onblur = () => {
        clearTimeout(timeout);
        setScrollTop(bfscrolltop);
      };
    }
  }
```

## 点击双击事件并存

```html
<template>
  <div class="touch-wrapper" @click.stop="onTap">
    <slot></slot>
  </div>
</template>

<script>
let lastClickTime = 0;
let clickTimer;

export default {
  name: 'BaseTouch',
  props: {
    // 单击事件延迟时间
    delay: {
      type: Number,
      default: 300
    }
  },
  methods: {
    onTap(e) {
      const nowTime = new Date().getTime();
      // 在规定时间内再触发点击事件时执行doubleTap，即此时为双击事件，否则为单击事件
      if (nowTime - lastClickTime < this.delay) {
        lastClickTime = 0;
        if (clickTimer) {
          clearTimeout(clickTimer);
        }
        this.$emit('double-tap', e);
      } else {
        lastClickTime = nowTime;
        clickTimer = setTimeout(() => {
          this.$emit('tap', e);
        }, this.delay);
      }
    }
  }
};
</script>
```
