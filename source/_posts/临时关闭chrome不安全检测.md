---
title: 临时关闭chrome不安全检测
date: 2019-04-07 15:18:52
tags:
---

有时候在使用chrome做开发的时候需要临时关闭一些安全检测

例如在获取定位的时候因为本地调试域名没有https所以无法调用 navigator.geolocation.getCurrentPosition 方法

1. 打开 chrome://flags/#unsafely-treat-insecure-origin-as-secure 

2. 把 Insecure origins treated as secure 设置为 enabled

3. 在 输入框中填入 本地测试域名即可 