---
layout: post
title: linux C语言printf实现光标隐藏
date: 2018-03-27 09:09:00 +0800
category: 学习记录
---

## 终端不显示光标

```c
printf("\033[?25l");
```

## 恢复光标显示

```c
printf("\033[?25h");
```

