---
title: Sentinel源码解析一
tags:
  - Sentinel
  - 并发
categories:
  - java框架
  - Sentinel
date: 2019-10-11 19:42:31
---

## 引言
`Sentinel`作为ali开源的一款轻量级流控框架，**主要以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度来帮助用户保护服务的稳定性**。相比于`Hystrix`，`Sentinel`的设计更加简单，在 `Sentinel`中资源定义和规则配置是分离的，也就是说用户可以先通过`Sentinel API`给对应的业务逻辑定义资源（埋点），然后在需要的时候再配置规则，通过这种组合方式，极大的增加了`Sentinel`流控的灵活性。

引入`Sentinel`带来的性能损耗非常小。只有在业务单机量级超过25W QPS的时候才会有一些显著的影响（5% - 10% 左右），单机QPS不太大的时候损耗几乎可以忽略不计。

`Sentinel`提供两种埋点方式:

* `try-catch` 方式（通过 `SphU.entry(...)`），用户在 catch 块中执行异常处理 / fallback
* `if-else` 方式（通过 `SphO.entry(...)`），当返回 false 时执行异常处理 / fallback

## 写在前面

在此之前，需要先了解一下`Sentinel`的工作流程
在 `Sentinel` 里面，所有的资源都对应一个资源名称（`resourceName`），每次资源调用都会创建一个 `Entry` 对象。`Entry` 可以通过对主流框架的适配自动创建，也可以通过注解的方式或调用 `SphU API` 显式创建。`Entry` 创建的时候，同时也会创建一系列功能插槽（`slot chain`），这些插槽有不同的职责，例如默认情况下会创建一下7个插槽：

* `NodeSelectorSlot` 负责收集资源的路径，并将这些资源的调用路径，以树状结构存储起来，用于根据调用路径来限流降级；
* `ClusterBuilderSlot` 则用于存储资源的统计信息以及调用者信息，例如该资源的 `RT, QPS, thread count` 等等，这些信息将用作为多维度限流，降级的依据；
* `StatisticSlot` 则用于记录、统计不同纬度的 `runtime` 指标监控信息；
* `FlowSlot` 则用于根据预设的限流规则以及前面 `slot` 统计的状态，来进行流量控制；
* `AuthoritySlot` 则根据配置的黑白名单和调用来源信息，来做黑白名单控制；
* `DegradeSlot` 则通过统计信息以及预设的规则，来做熔断降级；
* `SystemSlot` 则通过系统的状态，例如 `load1` 等，来控制总的入口流量

注意：这里的插槽链都是一一对应资源名称的

上面的所介绍的插槽（`slot chain`）是`Sentinel`非常重要的概念。同时还有一个非常重要的概念那就是`Node`，为了帮助理解，尽我所能画了下面这张图，可以看到整个结构非常的像一棵树：

![](http://img.souche.com/f2e/f88669970def3f8da53f6488b3a31366.jpg)

简单解释下上图：

* 顶部蓝色的`node`节点为根节点，全局唯一
* 下面黄色的节点为入口节点，每个`CentextName`(上下文名称)一一对应一个
	* 可以有多个子节点（对应多种资源）
* 中间绿色框框中的节点都是属于同一个资源的(相同的`ResourceName`)
* 最底下紫色的节点是集群节点，可以理解成绿色框框中Node资源的整合

以上2个概念务必要理清楚，之后再一步一步看源码会比较清晰

下面我们将从入口源码开始一步一步分析整个调用过程：

## 源码分析

下面的是一个`Sentinel`使用的示例代码，我们就从这里切入开始分析

```java
// 创建一个名称为entrance1，来源为appA 的上下文Context
ContextUtil.enter("entrance1", "appA");
// 创建一个资源名称nodeA的Entry
 Entry nodeA = SphU.entry("nodeA");
 if (nodeA != null) {
    nodeA.exit();
 }
 // 清除上下文
 ContextUtil.exit();
```

### ContextUtil.enter("entrance1", "appA")

```java
public static Context enter(String name, String origin) {
	  // 判断上下文名称是否为默认的名称（sentinel_default_context） 是的话直接抛出异常
    if (Constants.CONTEXT_DEFAULT_NAME.equals(name)) {
        throw new ContextNameDefineException(
            "The " + Constants.CONTEXT_DEFAULT_NAME + " can't be permit to defined!");
    }
    return trueEnter(name, origin);
}

protected static Context trueEnter(String name, String origin) {
	  // 先从ThreadLocal中尝试获取，获取到则直接返回
    Context context = contextHolder.get();
    if (context == null) {
        Map<String, DefaultNode> localCacheNameMap = contextNameNodeMap;
        // 尝试从缓存中获取该上下文名称对应的 入口节点
        DefaultNode node = localCacheNameMap.get(name);
        if (node == null) {
      		 // 判断缓存中入口节点数量是否大于2000
            if (localCacheNameMap.size() > Constants.MAX_CONTEXT_NAME_SIZE) {
                setNullContext();
                return NULL_CONTEXT;
            } else {
                try {
                		// 加锁
                    LOCK.lock();
                    // 双重检查锁
                    node = contextNameNodeMap.get(name);
                    if (node == null) {
                    	 // 判断缓存中入口节点数量是否大于2000
                        if (contextNameNodeMap.size() > Constants.MAX_CONTEXT_NAME_SIZE) {
                            setNullContext();
                            return NULL_CONTEXT;
                        } else {
                            // 根据上下文名称生成入口节点（entranceNode）
                            node = new EntranceNode(new StringResourceWrapper(name, EntryType.IN), null);
                            // 加入至全局根节点下
                            Constants.ROOT.addChild(node);
                            // 加入缓存中
                            Map<String, DefaultNode> newMap = new HashMap<>(contextNameNodeMap.size() + 1);
                            newMap.putAll(contextNameNodeMap);
                            newMap.put(name, node);
                            contextNameNodeMap = newMap;
                        }
                    }
                } finally {
                    LOCK.unlock();
                }
            }
        }
        // 初始化上下文对象
        context = new Context(node, name);
        context.setOrigin(origin);
        // 设置到当前线程中
        contextHolder.set(context);
    }

    return context;
}
```
主要做了2件事情

1. 根据`ContextName`生成`entranceNode`，并加入缓存，每个`ContextName`对应一个入口节点`entranceNode`
2. 根据`ContextName`和`entranceNode`初始化上下文对象，并将上下文对象设置到当前线程中

这里有几点需要注意：

1. 入口节点数量不能大于2000，大于会直接抛异常
2. 每个`ContextName`对应一个入口节点`entranceNode`
3. 每个`entranceNode`都有共同的父节点。也就是根节点

### Entry nodeA = SphU.entry("nodeA")

```java
// SphU.class
public static Entry entry(String name) throws BlockException {
    // 默认为 出口流量类型，单位统计数为1
    return Env.sph.entry(name, EntryType.OUT, 1, OBJECTS0);
}

// CtSph.class
public Entry entry(String name, EntryType type, int count, Object... args) throws BlockException {
    // 生成资源对象
    StringResourceWrapper resource = new StringResourceWrapper(name, type);
    return entry(resource, count, args);
}
public Entry entry(ResourceWrapper resourceWrapper, int count, Object... args) throws BlockException {
    return entryWithPriority(resourceWrapper, count, false, args);
}
```
上面的代码比较简单，不指定`EntryType`的话，则默认为出口流量类型，最终会调用`entryWithPriority`方法，主要业务逻辑也都在这个方法中

* entryWithPriority方法

```java
private Entry entryWithPriority(ResourceWrapper resourceWrapper, int count, boolean prioritized, Object... args)
    throws BlockException {
    // 获取当前线程上下文对象
    Context context = ContextUtil.getContext();
    // 上下文名称对应的入口节点是否已经超过阈值2000，超过则会返回空 CtEntry
    if (context instanceof NullContext) {
        return new CtEntry(resourceWrapper, null, context);
    }

    if (context == null) {
        // 如果没有指定上下文名称，则使用默认名称，也就是默认入口节点
        context = InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME);
    }

    // 全局开关
    if (!Constants.ON) {
        return new CtEntry(resourceWrapper, null, context);
    }
    // 生成插槽链
    ProcessorSlot<Object> chain = lookProcessChain(resourceWrapper);

    /*
     * 表示资源(插槽链)超过6000，因此不会进行规则检查。
     */
    if (chain == null) {
        return new CtEntry(resourceWrapper, null, context);
    }
    // 生成 Entry 对象
    Entry e = new CtEntry(resourceWrapper, chain, context);
    try {
        // 开始执行插槽链 调用逻辑
        chain.entry(context, resourceWrapper, null, count, prioritized, args);
    } catch (BlockException e1) {
        // 清除上下文
        e.exit(count, args);
        throw e1;
    } catch (Throwable e1) {
        // 除非Sentinel内部存在错误，否则不应发生这种情况。
        RecordLog.info("Sentinel unexpected exception", e1);
    }
    return e;
}
```
这个方法可以说是涵盖了整个Sentinel的核心逻辑

1. 获取上下文对象，如果上下文对象还未初始化，则使用默认名称初始化。初始化逻辑在上文已经分析过
2. 判断全局开关
3. 根据给定的资源生成插槽链，插槽链是跟资源相关的，Sentinel最关键的逻辑也都在各个插槽中。初始化的逻辑在`lookProcessChain(resourceWrapper);`中，下文会分析
4. 依顺序执行每个插槽逻辑

### lookProcessChain(resourceWrapper)方法

`lookProcessChain`方法为指定资源生成插槽链，下面我们来看下它的初始化逻辑

```java
ProcessorSlot<Object> lookProcessChain(ResourceWrapper resourceWrapper) {
    // 根据资源尝试从全局缓存中获取
    ProcessorSlotChain chain = chainMap.get(resourceWrapper);
    if (chain == null) {
        // 非常常见的双重检查锁
        synchronized (LOCK) {
            chain = chainMap.get(resourceWrapper);
            if (chain == null) {
                // 判断资源数是否大于6000
                if (chainMap.size() >= Constants.MAX_SLOT_CHAIN_SIZE) {
                    return null;
                }
                // 初始化插槽链
                chain = SlotChainProvider.newSlotChain();
                Map<ResourceWrapper, ProcessorSlotChain> newMap = new HashMap<ResourceWrapper, ProcessorSlotChain>(
                    chainMap.size() + 1);
                newMap.putAll(chainMap);
                newMap.put(resourceWrapper, chain);
                chainMap = newMap;
            }
        }
    }
    return chain;
}
```
1. 根据资源尝试从全局缓存中获取插槽链。每个资源对应一个插槽链（资源嘴多只能定义6000个）
2. 初始化插槽链上的插槽（`SlotChainProvider.newSlotChain()`方法中）

下面我们看下初始化插槽链上的插槽的逻辑

### SlotChainProvider.newSlotChain()

```java
public static ProcessorSlotChain newSlotChain() {
    // 判断是否已经初始化过
    if (builder != null) {
        return builder.build();
    }
	  // 加载 SlotChain 
    resolveSlotChainBuilder();
    // 加载失败则使用默认 插槽链 
    if (builder == null) {
        RecordLog.warn("[SlotChainProvider] Wrong state when resolving slot chain builder, using default");
        builder = new DefaultSlotChainBuilder();
    }
    // 构建完成
    return builder.build();
}

/**
 * java自带 SPI机制 加载 slotChain
 */
private static void resolveSlotChainBuilder() {
    List<SlotChainBuilder> list = new ArrayList<SlotChainBuilder>();
    boolean hasOther = false;
    // 尝试获取自定义SlotChainBuilder，通过JAVA SPI机制扩展
    for (SlotChainBuilder builder : LOADER) {
        if (builder.getClass() != DefaultSlotChainBuilder.class) {
            hasOther = true;
            list.add(builder);
        }
    }
    if (hasOther) {
        builder = list.get(0);
    } else {
        // 未获取到自定义 SlotChainBuilder 则使用默认的
        builder = new DefaultSlotChainBuilder();
    }

    RecordLog.info("[SlotChainProvider] Global slot chain builder resolved: "
        + builder.getClass().getCanonicalName());
}
```

1. 首先会尝试获取自定义的`SlotChainBuilder`来构建插槽链，自定义的`SlotChainBuilder`可以通过JAVA SPI机制来扩展
2. 如果未配置自定义的`SlotChainBuilder`，则会使用默认的`DefaultSlotChainBuilder`来构建插槽链，`DefaultSlotChainBuilder`所构建的插槽就是文章开头我们提到的7种`Slot`。每个插槽都有其对应的职责，各司其职，后面我们会详细分析这几个插槽的源码，及所承担的职责。

## 总结

文章开头的提到的两个点(插槽链和Node)，这是Sentinel的重点，理解这两点对于阅读源码来说事半功倍

Sentinel系列

* [Sentinel源码解析一](http://wangjunnan.club/2019/10/11/Sentinel%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E4%B8%80/)

* [Sentinel源码解析二（slot总览）](http://wangjunnan.club/2019/10/16/Sentinel%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E4%BA%8C%EF%BC%88slot%E6%80%BB%E8%A7%88%EF%BC%89/)

* [Sentinel源码解析二（滑动窗口流量统计）](http://wangjunnan.club/2019/10/25/Sentinel%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E4%B8%89%EF%BC%88%E6%BB%91%E5%8A%A8%E7%AA%97%E5%8F%A3%E6%B5%81%E9%87%8F%E7%BB%9F%E8%AE%A1%EF%BC%89/)

