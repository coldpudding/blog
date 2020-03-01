---
title: ios-iframe-宽度问题处理
date: 2020-01-22 16:00:48
tags:
---

###  问题原因
首先 IOS 的 iframe 会根据内容自动适应，现在手机端适配比较流行rem做法，会根据页面的宽度自动设置 html 的 font-size，所以就会出现一个情况

内容页面根据 iframe 宽度算出一个 font-size 以后，页面的宽度使用 rem 设置的，导致 宽度 被撑大了，然后 iframe 宽度 就会自动扩大，这个时候，内容页又会重新根据 iframe 宽度算一个新的值，成了一个死循环，让页面越来越大

### 处理办法

固定 iframe 的大小，单纯通过 css 是不行的，需要用到 dom 属性
```
<iframe height="100%" scrolling='no' style="width:1px; min-width:100%; *width:100%;" />
```