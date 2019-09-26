---
title: JAVA中时间的表现形式
tags:
  - java基础
categories:
  - java基础
date: 2019-09-26 16:02:02
---
## 引言
大多数时候我们在表示时间的概念时，可能并不会去特意关注时区这个概念，毕竟在国内时区都是统一的，我们只需要使用默认时区 北京时间 就可以了。是的，在写这篇文章之前，我也没有过多的去关注时区的概念，以及在JAVA中的表现形式。主要原因还是目前所在的公司是跨境公司，公司主要业务在印度，那么显然印度时间跟北京时间是不同的。

## 时区的概念

在这之前先说一下时区的概念，以及解释几个专有名词。
时区是地球上的区域使用同一个时间定义，整个全球被分成24个时区。所以每差一个时区，区时相差一个小时，相差多少个时区，就相差多少个小时。东边的时区时间比西边的时区时间早。北京时间值得就是东八区的时间。

### UTC时间和本地时间

UTC时间又称协调世界时，是最主要的世界时间标准，其以原子时秒长为基础，在时刻上尽量接近于格林尼治标准时间(GMT)。对于大多数用途来说，UTC时间被认为能与GMT时间互换，但GMT时间已不再被科学界所确定。
**如果时间是以协调世界时（UTC）表示，则在时间后面直接加上一个“Z”（不加空格）。“Z”是协调世界时中0时区的标志。**

* UTC偏移量和本地时间
例如北京时间的UTC偏移量是 +8，则当UTC时间是 2:00 时，北京时间的 10:00。所以借助于UTC时间，我们可以把全球的时间都统一起来。

### 时间戳

时区可以很好的表示各地的时间，但各个时区的字面时间显示仍然还是不同的，所以我们需要一种方式在来让世界各个角落的时间都有一样的表现形式。这就是时间戳，它也被称为Unix时间(Unix time)，定义为从格林威治时间1970年01月01日00时00分00秒起至现在的总秒数。


## Java8中的时间类型

在JAVA8中，提供了新的时间日期操作类，相对于之前的 Date 类，可以说使用起来方便了许多，下面依次来介绍几个关键类。

* Instant 用来表示时间戳
* LocalDateTime 单纯表示字面时间，不带时区信息。也就是说 `LocalDateTime = 2019-09-25 00:00:00`，这个时间我们并不知道是北京时间还是东京时间，仅仅用作字面时间
* ZonedDateTime 含有时区信息的时间，类似于北京时间，UTC+8

### 它们如何表示时间

```java
System.out.println(LocalDateTime.now());
System.out.println(Instant.now());
System.out.println(ZonedDateTime.now());

输出
2019-09-26T14:50:43.741
2019-09-26T06:50:43.742Z
2019-09-26T14:50:43.803+08:00[Asia/Shanghai]
```
可以看到 
`LocalDateTime` 会根据UTC时间获取到本地时间显示，但是会把时区信息丢弃
`Instant` 则直接表示的UTC世界标准时间
`ZonedDateTime` 会根据UTC时间获取到本地时间显示，同时也会显示当前的时区信息

其实ZonedDateTime 是由 Instant 加上时区信息结合而来，通过ZonedDateTime可以直接获取到Instant时间戳信息

### 如何转换

转换的核心其实都是往时间戳（世界标准时间）靠拢，试想是不是这样？只有转成了标准时间才能转成其他时区的时间

* `ZonedDateTime` <-> `Instant`

```java
// zonedDateTime -> instant
ZonedDateTime zonedDateTime = ZonedDateTime.now();
Instant instant = zonedDateTime.toInstant();

// instant -> zonedDateTime 要指定时区信息
ZoneId DEFAULT_ZONE_ID = ZoneId.of("Asia/Shanghai");
zonedDateTime = instant.atZone(DEFAULT_ZONE_ID);
```

* `LocalDateTime` <-> `Instant`

```java
// LocalDateTime -> Instant
LocalDateTime localDateTime = LocalDateTime.now();
Instant instant = localDateTime.toInstant(ZoneOffset.of("+08:00"));

// Instant -> ZonedDateTime -> LocalDateTime
// Instant 需要先转换成 ZonedDateTime 再转换成本地时间
ZonedDateTime zonedDateTime = instant.atZone(ZoneId.of("Asia/Shanghai"));
localDateTime = zonedDateTime.toLocalDateTime();
```

* `LocalDateTime` <-> `ZonedDateTime`

```java
// LocalDateTime -> ZonedDateTime 需要指定本地所在时区
LocalDateTime localDateTime = LocalDateTime.now();
ZonedDateTime zonedDateTime = localDateTime.atZone(ZoneId.of("Asia/Shanghai"));

// 直接转换成本地时间
localDateTime = zonedDateTime.toLocalDateTime();
```
可以看到，在转换过程中，会频繁的手动指定时区，主要原因是`LocalDateTime`并不带有时区信息，如果我们要转换成标准时间，就需要手动指定时区。**所以为了避免转换过程中的错误，我们应该尽量使用时间戳来来传输时间。**

#### 与其他api的转换

* Timestamp <-> LocalDateTime

```
// Timestamp -> LocalDateTime
Timestamp timestamp = Timestamp.from(Instant.now());
LocalDateTime localDateTime = timestamp.toLocalDateTime();

// LocalDateTime -> Timestamp 有两种方式
// 1. 直接按本地默认时区转时区
timestamp = Timestamp.valueOf(localDateTime);
// 2. 指定时区转
timestamp = Timestamp.from(localDateTime.toInstant(ZoneOffset.of("+08:00")));

```

* Date <-> LocalDateTime

```java
// Date -> LocalDateTime
Date date = new Date();
LocalDateTime localDateTime = date.toInstant().atZone(ZoneId.of("Asia/Shanghai")).toLocalDateTime();

// LocalDateTime -> Date
date = Date.from(localDateTime.toInstant(ZoneOffset.of("+08:00")));
```

更多例子[DateTimeUtils.java](https://github.com/WangJunnan/walm-common/blob/master/src/main/java/com/walm/common/util/DateTimeUtils.java)

## 总结

最后还是建议操作时间时使用时间戳，这样子没有时区的概念，也就不会产生歧义。






