---
layout: post
title: 设计模式实战
description: 
date: 2021-08-07
---

# 组合模式

组合模式（Composite Pattern），又叫部分整体模式，是用于把一组相似的对象当作一个单一的对象。

1. 意图：将对象组合成树形结构以表示"部分-整体"的层次结构。组合模式使得用户对单个对象和组合对象的使用具有一致性。
2. 主要解决：它在我们树型结构的问题中，模糊了简单元素和复杂元素的概念，客户程序可以像处理简单元素一样来处理复杂元素，从而使得客户程序与复杂元素的内部结构解耦。
3. 何时使用： 1、您想表示对象的部分-整体层次结构（树形结构）。 2、您希望用户忽略组合对象与单个对象的不同，用户将统一地使用组合结构中的所有对象。

## 答题卡制作工具

* 场景

答题卡的基本结构元件：定位块、标题、二维码、准考证号、缺考标记、单选题、解答题

<img src="../../../assets/images/design-pattern-actions/answersheet.png">

答题卡制作工具有两个核心任务：
1. 以html的形式来呈现整个答题卡的结构
2. 提供答题卡各结构元件相对于页面的精准位置信息

* 抽象

1. 答题卡中的每个结构元件都可以用一个div来呈现
2. 结构元件会有父子的层级关系
3. 使用css的绝对定位来布局各个结构元件
4. 每个结构元件负责自己的html生成，所有结构元件组织成父子的层级关系，通过递归层级调用的方式就可以生成用于呈现整个答题卡的html。这就是一种化繁为简的思想。
5. 每个结构元件都相对于父亲进行绝对定位，基于上述的父子层级关系，通过递归层级计算，就可以很方便的得到各个结构元件相对于页面的精准位置信息了。

* 类设计

``` typescript

// 所有结构元件的基类
export class Element {
  top?: number;
  left?: number;
  width?: number;
  height?: number;

  pTop?: number;
  pLeft?: number;

  parent?: any;
  children?: any = [];

  elementType?: string = '';

  constructor(top: number, left: number, width: number, height: number) {
    this.top = top;
    this.left = left;
    this.width = width;
    this.height = height;
  }

  writeHtmlBegin(answersheet: any): void {
  }

  protected writeHtml(answersheet: any): void {
  }

  writeHtmlEnd(answersheet: any): void {
  }

  calcLocation(): void {
  }
}

// 答题卡的一面纸张
export class PageBox extends Element {
  constructor(top: number, left: number, width: number, height: number) {
    super(top, left, width, height);
    this.elementType = 'PageBox';
  }
}

// 定位点
export class LocatePoint extends Element {
  constructor(top: number, left: number, width: number, height: number) {
    super(top, left, width, height);
    this.elementType = 'LocatePoint';
  }
}

// 选择题
export class ChoiceBox extends Element {
  constructor(top: number, left: number, width: number, height: number) {
    super(top, left, width, height);
    this.elementType = 'ChoiceBox';
  }
}

```

## 客观题填涂识别

* 场景

<img src="../../../assets/images/design-pattern-actions/object-rec.png">

1. 前提：已能准确获得客观题每一区块、每一个选项的图片。
2. 填涂识别的任务是判断每一个选项是否填涂。
3. 理论上只需要计算出每一个选项的平均灰度值和填涂面积，然后设置一个阈值，就可以判断出选项是否填涂。
4. 现实场景要复杂很多，例如填涂较淡、擦除、印刷问题等，根本就不是一个阈值能搞定。
5. 算法上考虑在一个题目范围内各个选项间进行对比，一个区块范围内相同选项间进行对比，整个答题卡范围内相同选项间进行对比，从而使填涂识别具有更强的适应性

* 抽象

1. 由于需要进行横向、纵向的对比分析，答题卡中的客观题可抽象成三个结构元件：客观题区块、客观题、选项
2. 三个结构元件组成父子的层级关系，相互关联，随时可访问
3. 如此组织代码，很容易实现题目范围内、区块范围内、整个答题卡范围内的对比

* 类设计

# 参考资料

[设计模式-菜鸟教程](https://www.runoob.com/design-pattern/design-pattern-tutorial.html)