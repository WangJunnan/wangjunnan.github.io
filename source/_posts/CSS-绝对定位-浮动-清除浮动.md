---
title: 'CSS 绝对定位 ,浮动,清除浮动'
tags:
  - CSS
categories:
  - 前端技术
date: 2016-09-21 15:47:10
---

# 首先：
我们需要知道div元素（块级元素）独占一行
```
<!DOCTYPE html>
<html>
<head>
</head>
<body>
<div id="1" style="background:red;height:200px;padding:5px;position:relative">
	<div id="2" style="background:green;height:50px;">box1</div>
	<!-- div元素独占一行 -->
	<div id="3" style="background:pink;height:50px;">box2</div>
</div>

</body>
</html>
```
如下图所示，box1和box2独占一行，可见如果每个div都独占一行，**我们根本无法进行布局**，所以我们就需要绝对定位和浮动来帮助我们布局！
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-f5d62008959320c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 绝对定位的参照关系
绝对定位会以最近的设置了position属性的父级为参照物进行定位，如下图box2就以id为1 的div为父级参照进行定位

```
<!DOCTYPE html>
<html>
<head>
</head>
<body>
<div id="0" style="background:blue;padding:20px">
<div id="1" style="background:red;height:200px;padding:5px;position:relative">
	<div id="2" style="background:black;width:50px;height:50px;">box1</div>
	<!-- 绝对定位的父级参照物是最近的设置了position的值的父级元素，这里为 参照物为1 -->
	<div id="3" style="background:pink;width:50px;height:50px;position:absolute;left:50px;top:0px">box2</div>
</div>
</div>

</body>
</html>
```

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-6f6a363086e1988b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 浮动

>无论多么复杂的布局，其基本出发点均是：如何在一行显示多个div元素。显然标准流已经无法满足需求，这就要用到浮动。      
浮动可以理解为让某个div元素脱离标准流，漂浮在标准流之上，和标准流不是一个层次。

正常的div布局应该是像下图一样的

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-620f5a4c5d1bcb28.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


```
<body>
<div id="1" style="background:gray;height:500px;margin-top:5px">
	<div id="5" style="background:blue;width:100px;height:50px;">box1</div>
	<div id="5" style="background:yellow;width:60px;height:50px">box2</div>
	<div id="5" style="background:green;width:70px;height:70px;">box3</div>
	<div id="5" style="background:yellow;width:200px;height:50px;">box4</div>
</div>

</body>
```
假如我们将box2设置浮动 style中加上属性 float:left 向左浮动我们可以发现如下图的样子

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-b6aa14ab96ea5205.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其中box2漂浮在了box3上面，从而覆盖了一部分的box3，从而我们可以知道浮动的元素是脱离了标准流的！

之后我们将box3也设置为向左浮动，可以发现box3和box2并排了，并排在box2之后，而且box2和box3都漂浮在box4之上

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-8103251338c47eda.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
<div id="1" style="background:gray;height:500px;margin-top:5px">
	<div id="5" style="background:blue;width:100px;height:50px;">box1</div>
	<div id="5" style="background:yellow;width:60px;height:50px;float:left">box2</div>
	<div id="5" style="background:green;width:70px;height:70px;float:left">box3</div>
	<div id="5" style="background:pink;width:200px;height:90px;">box4</div>
</div>
```
# clear 清除浮动
清除浮动的关键字
clear : none | left | right | both
什么时候需要清除浮动呢？

以上面的例子 ，如果我们希望box4不被这些漂浮的元素所覆盖，需要怎么做呢？
这时候就需要清除浮动了 clear了
只需要在box4 添加CSS属性 clear:left ,就可以清除其左边的浮动元素影响，不被他们所覆盖

![加了clear之后的.png](http://upload-images.jianshu.io/upload_images/2717496-e6f377ffa996d32f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
<div id="1" style="background:gray;height:500px;margin-top:5px">
	<div id="5" style="background:blue;width:100px;height:50px;">box1</div>
	<div id="5" style="background:yellow;width:60px;height:50px;float:left">box2</div>
	<div id="5" style="background:green;width:70px;height:70px;float:left">box3</div>
	<div id="5" style="background:pink;width:200px;height:90px;clear:left">box4</div>
</div>
```
还有当我们浮动的元素不希望跟在其他浮动元素之后时，也可以使用clear 属性来另起一行

```
<div id="1" style="background:gray;height:500px;margin-top:5px">
	<div id="5" style="background:blue;width:100px;height:50px;">box1</div>
	<div id="5" style="background:yellow;width:60px;height:50px;float:left">box2</div>
	<div id="5" style="background:green;width:70px;height:70px;float:left;clear:left">box3</div>
</div>
```

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2717496-7372e799cf7e308a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
