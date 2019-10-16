---
title: Sentinel源码解析二（slot总览）
tags:
  - Sentinel
  - 并发
categories:
  - java框架
  - Sentinel
date: 2019-10-16 09:57:32
---
## 写在前面
 
本文继续来分析Sentinel的源码，上篇文章对Sentinel的调用过程做了深入分析，主要涉及到了两个概念：插槽链和Node节点。那么接下来我们就根据插槽链的调用关系来依次分析每个插槽（slot）的源码。

默认插槽链的调用顺序，以及每种类型Node节点的关系都在上面文章开头分析过 []()

## NodeSelectorSlot

```java
/**
 * 相同的资源但是Context不同，分别新建 DefaultNode，并以ContextName为key
 */
private volatile Map<String, DefaultNode> map = new HashMap<String, DefaultNode>(10);

public void entry(Context context, ResourceWrapper resourceWrapper, Object obj, int count, boolean prioritized, Object... args)
    throws Throwable {
    
    // 根据ContextName尝试获取DefaultNode
    DefaultNode node = map.get(context.getName());
    if (node == null) {
        synchronized (this) {
            node = map.get(context.getName());
            if (node == null) {
            // 初始化DefaultNode，每个Context对应一个
                node = new DefaultNode(resourceWrapper, null);
                HashMap<String, DefaultNode> cacheMap = new HashMap<String, DefaultNode>(map.size());
                cacheMap.putAll(map);
                cacheMap.put(context.getName(), node);
                map = cacheMap;
            }
            // 构建 Node tree
            ((DefaultNode)context.getLastNode()).addChild(node);
        }
    }

    context.setCurNode(node);
    // 唤醒执行下一个插槽
    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```
`NodeSelectorSlot`顾名思义是用来构建`Node`的。
我们可以看到`NodeSelectorSlot`对于不同的上下文都会生成一个`DefaultNode`。这里还有一个要注意的点：**相同的资源({@link ResourceWrapper#equals(Object)})将全局共享相同的{@link ProcessorSlotChain}，无论在哪个上下文中**，因此不同的上下文可以进入到同一个对象的`NodeSelectorSlot.entry`方法中，那么这里要怎么区分不同的上下文所创建的资源Node呢？显然可以使用上下文名称作为映射键以区分相同的资源Node。

然后这里要考虑另一个问题。一个资源有可能创建多个`DefaultNode`（有多个上下文时），那么我们应该如何快速的获取总的统计数据呢？

答案就在下一个Slot(`ClusterBuilderSlot`)中被解决了。

## ClusterBuilderSlot

上面有提到一个问题，我们要如何统计不同上下文相同资源的总量数据。`ClusterBuilderSlot`给了很好的解决方案：具有相同资源名称的共享一个`ClusterNode`。

```java
// 相同的资源共享一个 ClusterNode
private static volatile Map<ResourceWrapper, ClusterNode> clusterNodeMap = new HashMap<>();

private static final Object lock = new Object();

private volatile ClusterNode clusterNode = null;

@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                  boolean prioritized, Object... args)
    throws Throwable {
    // 判断本资源是否已经初始化过clusterNode
    if (clusterNode == null) {
        synchronized (lock) {
            if (clusterNode == null) {
                // 没有初始化则初始化clusterNode
                clusterNode = new ClusterNode(resourceWrapper.getName(), resourceWrapper.getResourceType());
                HashMap<ResourceWrapper, ClusterNode> newMap = new HashMap<>(Math.max(clusterNodeMap.size(), 16));
                newMap.putAll(clusterNodeMap);
                newMap.put(node.getId(), clusterNode);

                clusterNodeMap = newMap;
            }
        }
    }
    // 给相同资源的DefaultNode设置一样的ClusterNode
    node.setClusterNode(clusterNode);

    /*
     * 如果有来源则新建一个来源Node
     */
    if (!"".equals(context.getOrigin())) {
        Node originNode = node.getClusterNode().getOrCreateOriginNode(context.getOrigin());
        context.getCurEntry().setOriginNode(originNode);
    }

    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```
上面的代码其实就是做了一件事情，为资源创建`CluserNode`。这里我又要提一嘴 **相同的资源({@link ResourceWrapper#equals(Object)})将全局共享相同的{@link ProcessorSlotChain}，无论在哪个上下文中**。也就是说，能进入到同一个`ClusterBuilderSlot`对象的`entry`方法的请求都是来自同一个资源的，所以这些相同资源需要初始化一个统一的`CluserNode`用来做流量的汇总统计。

## LogSlot
代码比较简单，逻辑就是打印异常日志，就不分析了

## StatisticSlot

`StatisticSlot` 是 `Sentinel` 的核心功能插槽之一，用于统计实时的调用数据。

```java
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                  boolean prioritized, Object... args) throws Throwable {
    try {
        // 先进行后续的check，包括规则的check，黑白名单check
        fireEntry(context, resourceWrapper, node, count, prioritized, args);

        // 统计默认qps 线程数
        node.increaseThreadNum();
        node.addPassRequest(count);

        if (context.getCurEntry().getOriginNode() != null) {
            // 根据来源统计qps 线程数
            context.getCurEntry().getOriginNode().increaseThreadNum();
            context.getCurEntry().getOriginNode().addPassRequest(count);
        }

        if (resourceWrapper.getEntryType() == EntryType.IN) {
            // 统计入口 qps 线程数
            Constants.ENTRY_NODE.increaseThreadNum();
            Constants.ENTRY_NODE.addPassRequest(count);
        }

        .... 省略其他代码
    }
}
```
## StatisticSlot主要做了4种不同维度的流量统计

1. 资源在上下文维度（DefaultNode）的统计
2. clusterNode 维度的统计
3. Origin 来源维度的统计
4. 入口全局流量的统计

关于流量的统计原理的本文就不深入分析了，接下来的文章中会单独分析

## SystemSlot

`SystemSlot`比较简单，其实就是根据`StatisticSlot`所统计的全局入口流量进行限流。


## AuthoritySlot

```java
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count, boolean prioritized, Object... args)
    throws Throwable {
    checkBlackWhiteAuthority(resourceWrapper, context);
    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}

void checkBlackWhiteAuthority(ResourceWrapper resource, Context context) throws AuthorityException {
    Map<String, Set<AuthorityRule>> authorityRules = AuthorityRuleManager.getAuthorityRules();

    if (authorityRules == null) {
        return;
    }
	  // 根据资源获取黑白名单规则
    Set<AuthorityRule> rules = authorityRules.get(resource.getName());
    if (rules == null) {
        return;
    }
    // 对规则进行校验，只要有一条不通过 就抛异常
    for (AuthorityRule rule : rules) {
        if (!AuthorityRuleChecker.passCheck(rule, context)) {
            throw new AuthorityException(context.getOrigin(), rule);
        }
    }
}
```
`AuthoritySlot`会对资源的黑白名单做检查，并且只要有一条不通过就抛异常。

## FlowSlot

```java
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                  boolean prioritized, Object... args) throws Throwable {
    checkFlow(resourceWrapper, context, node, count, prioritized);

    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}

void checkFlow(ResourceWrapper resource, Context context, DefaultNode node, int count, boolean prioritized)
    throws BlockException {
    // 检查限流规则
    checker.checkFlow(ruleProvider, resource, context, node, count, prioritized);
}
```

这个`slot`主要根据预设的资源的统计信息，按照固定的次序，依次生效。如果一个资源对应两条或者多条流控规则，则会根据如下次序依次检验，直到全部通过或者有一个规则生效为止:
并且同样也会根据三种不同的维度来进行限流：

1. 资源在上下文维度（DefaultNode）的统计
2. clusterNode 维度的统计
3. Origin 来源维度的统计

关于流控规则源码的深入分析就不在本篇文章赘述了

## DegradeSlot

这个`slot`主要针对资源的平均响应时间（RT）以及异常比率，来决定资源是否在接下来的时间被自动熔断掉。

## 总结

1. 相同的资源({@link ResourceWrapper#equals(Object)})将全局共享相同的{@link ProcessorSlotChain}，无论在哪个上下文中
2. 流控有多个维度，分别包括：1.不同上下文中的资源 2.相同资源 3.入口流量 3.相同的来源



