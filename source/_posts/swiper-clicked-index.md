---
title: swiper loop 循环模式下 获取 点击的索引
date: 2020-01-22 16:17:14
tags:
---

swiper 循环模式 通过复制多一些节点来实现循环的效果，并且多复制出来的节点不会绑定 vue 的 @click 事件，用户滑动完毕以后，或者滑动之前(具体没看源码)瞬间跳转到一个靠中间位置的对应索引上， swiper 提供 了几个属性来获取 index

1. realIndex 当前激活的 索引，不受循环模式的影响
2. activeIndex 跟上面也是一样的，当前激活的 索引，受循环模式影响，返回的并不是真实的 index，而是加上了复制的节点后的 index
3. clickedIndex 最后点击的索引，但是也是受循环模式影响

一般情况下并没啥问题，现在需要点击非激活的块，并且得到点击的 realIndex，这里就会发现上面的上面的办法好像都不行

### 方案一
将 clickedIndex 转换 成 realIndex

```js
...
on: {
    click() {
    	// 前面添加的 slide 数量
    	const prependSlidesCount =  ( this.slides.length - dataList.length ) / 2
        const clickedRealIndex = (this.clickedIndex + dataList.length - prependSlidesCount ) % dataList.length
    }
}
...
```

### 方案二
通过观察 复制的 节点发现，每个节点 有个 data-set 属性, 这个值也是真实的 index
```js
...
on: {
    click() {
    
        const clickedRealIndex = parseInt(
            this.clickedSlide.dataset.swiperSlideIndex || this.clickedIndex
        )

        或
        
        const clickedRealIndex = parseInt(
            this.clickedSlide.getAttribute('data-swiper-slide-index') || this.clickedIndex
        )
    }
}
...
```
