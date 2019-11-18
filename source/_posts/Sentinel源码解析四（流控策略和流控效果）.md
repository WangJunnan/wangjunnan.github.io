---
title: Sentinel源码解析四（流控策略和流控效果）
tags:
  - Sentinel
  - 中间件
categories:
  - Sentinel
  - 并发
date: 2019-11-18 10:50:25
---
## 引言

在分析Sentinel的上一篇文章中，我们知道了它是基于滑动窗口做的流量统计，那么在当我们能够根据流量统计算法拿到流量的实时数据后，下一步要做的事情自然就是基于这些数据做流控。在介绍`Sentinel`的流控模型之前，我们先来简单看下 Sentinel 后台是如何去定义一个流控规则的

![](http://img.souche.com/f2e/71089fbdf4dc7ab9b2b0afc01b99baf7.jpg)

对于上图的配置`Sentinel`把它抽象成一个`FlowRule`类，与其属性一一对应

* resource 资源名
* limitApp 限流来源，默认为default不区分来源
* grade    限流类型，有QPS和并发线程数两种类型
* count    限流阈值
* strategy 流控策略  1. 直接 2. 关联 3.链路 
* controlBehavior   流控效果 1.快速失败 2.预热启动 3.排队等待 4. 预热启动排队等待
* warmUpPeriodSec   流控效果为预热启动时的预热时长（秒）
* maxQueueingTimeMs 流控效果为排队等待时的等待时长 （毫秒）


下面我们来看下选择流控策略和流控效果的核心代码

```java
private static boolean passLocalCheck(FlowRule rule, Context context, DefaultNode node, int acquireCount,boolean prioritized) {
// 根据流控策略选择需要流控的Node维度节点                              
    Node selectedNode = selectNodeByRequesterAndStrategy(rule, context, node);
    if (selectedNode == null) {
        return true;
    }
    // 获取配置的流控效果 控制器 （1. 直接拒绝 2. 预热启动 3. 排队 4. 预热启动排队等待）
    return rule.getRater().canPass(selectedNode, acquireCount, prioritized);
}
```
上面的代码比较简单流程也很清晰，首先根据我们配置的流控策略获取到合适维度的 Node 节点（Node节点是Sentinel做流量统计的基本单位），然后再获取到规则中配置的流控效果控制器（1. 直接拒绝 2. 预热启动 3. 排队等待 4.预热启动排队等待）。

## 流控策略

下面我们来看下选择流控策略的源码分析

```java
static Node selectNodeByRequesterAndStrategy(/*@NonNull*/ FlowRule rule, Context context, DefaultNode node) {
    // 获取限流来源 limitApp
    String limitApp = rule.getLimitApp();
    // 获取限流策略
    int strategy = rule.getStrategy();
    // 获取当前 上下文的 来源
    String origin = context.getOrigin();
    
    // 如果规则配置的限流来源 limitApp 等于 当前上下文来源
    if (limitApp.equals(origin) && filterOrigin(origin)) {
    // 且配置的流控策略是 直接关联策略
        if (strategy == RuleConstant.STRATEGY_DIRECT) {
            // 直接返回当前来源 origin 节点
            return context.getOriginNode();
        }
		   // 配置的策略为关联或则链路
        return selectReferenceNode(rule, context, node);
        
        
    // 如果规则配置的限流来源 limitApp 等于 default 
    } else if (RuleConstant.LIMIT_APP_DEFAULT.equals(limitApp)) {
    // 且配置的流控策略是 直接关联策略
        if (strategy == RuleConstant.STRATEGY_DIRECT) {
            // 直接返回当前资源的 clusterNode
            return node.getClusterNode();
        }
        // 配置的策略为关联或则链路
        return selectReferenceNode(rule, context, node);
        
        // 如果规则配置的限流来源 limitApp 等于 other，且当前上下文origin不在流控规则策略中
    } else if (RuleConstant.LIMIT_APP_OTHER.equals(limitApp)
        && FlowRuleManager.isOtherOrigin(origin, rule.getResource())) {
        // 且配置的流控策略是 直接关联策略
        if (strategy == RuleConstant.STRATEGY_DIRECT) {
            return context.getOriginNode();
        }
        // 配置的策略为关联或则链路
        return selectReferenceNode(rule, context, node);
    }

    return null;
}

static Node selectReferenceNode(FlowRule rule, Context context, DefaultNode node) {
// 关联资源名称 （如果策略是关联 则是关联的资源名称，如果策略是链路 则是上下文名称）
    String refResource = rule.getRefResource();
    int strategy = rule.getStrategy();

    if (StringUtil.isEmpty(refResource)) {
        return null;
    }
    // 策略是关联
    if (strategy == RuleConstant.STRATEGY_RELATE) {
    // 返回关联的资源ClusterNode
        return ClusterBuilderSlot.getClusterNode(refResource);
    }
    // 策略是链路
    if (strategy == RuleConstant.STRATEGY_CHAIN) {
    // 当前上下文名称不是规则配置的name 直接返回null
        if (!refResource.equals(context.getName())) {
            return null;
        }
        return node;
    }
    // No node.
    return null;
}
```
这段代码的逻辑判断比较多，我们稍微理一下整个过程

* `LimitApp`的作用域只在配置的流控策略为`RuleConstant.STRATEGY_DIRECT`（直接关联）时起作用。其有三种配置，分别为`default`，`origin_name`，`other`
	* default 如果配置为default，表示统计不区分来源，当前资源的任何来源流量都会被统计（其实就是选择 Node 为 clusterNode 维度）
	* origin_name 如果配置为指定名称的 origin_name，则只会对当前配置的来源流量做统计
	* other 如果配置为other 则会对其他全部来源生效**但不包括第二条配置的来源**
* 当策略配置为 RuleConstant.STRATEGY_RELATE 或 RuleConstant.STRATEGY_CHAIN 时
	* STRATEGY_RELATE 关联其他的指定资源，如资源A想以资源B的流量状况来决定是否需要限流，这时资源A规则配置可以使用 STRATEGY_RELATE 策略
	* STRATEGY_CHAIN 对指定入口的流量限流，因为流量可以有多个不同的入口（EntranceNode）
* 对于上面几个节点之间的关系不清楚的可以去看我这篇文章开头的总览图 [Sentinel源码解析一（流程总览）](https://www.cnblogs.com/taromilk/p/11750962.html)

## 流控效果

关于流控效果的配置有四种，我们来看下它们的初始化代码

```java
/**
 * class com.alibaba.csp.sentinel.slots.block.flow.FlowRuleUtil
 */
private static TrafficShapingController generateRater(/*@Valid*/ FlowRule rule) {
// 只有Grade为统计 QPS时 才可以选择除默认流控效果外的 其他流控效果控制器
    if (rule.getGrade() == RuleConstant.FLOW_GRADE_QPS) {
        switch (rule.getControlBehavior()) {
        // 预热启动
            case RuleConstant.CONTROL_BEHAVIOR_WARM_UP:
                return new WarmUpController(rule.getCount(), rule.getWarmUpPeriodSec(),
                    ColdFactorProperty.coldFactor);
            // 超过 阈值 排队等待 控制器
            case RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER:
                return new RateLimiterController(rule.getMaxQueueingTimeMs(), rule.getCount());
            case RuleConstant.CONTROL_BEHAVIOR_WARM_UP_RATE_LIMITER:
            // 上面两个的结合体
                return new WarmUpRateLimiterController(rule.getCount(), rule.getWarmUpPeriodSec(),
                    rule.getMaxQueueingTimeMs(), ColdFactorProperty.coldFactor);
            case RuleConstant.CONTROL_BEHAVIOR_DEFAULT:
            default:
                // Default mode or unknown mode: default traffic shaping controller (fast-reject).
        }
    }
    // 默认控制器 超过 阈值 直接拒绝
    return new DefaultController(rule.getCount(), rule.getGrade());
}
```
可以比较清晰的看到总共对应有四种流控器的初始化

### 直接拒绝

```java
@Override
public boolean canPass(Node node, int acquireCount, boolean prioritized) {
// 获取当前qps
    int curCount = avgUsedTokens(node);
    // 判断是否已经大于阈值
    if (curCount + acquireCount > count) {
    // 如果当前流量具有优先级，则会提前去获取未来的通过资格
        if (prioritized && grade == RuleConstant.FLOW_GRADE_QPS) {
            long currentTime;
            long waitInMs;
            currentTime = TimeUtil.currentTimeMillis();
            waitInMs = node.tryOccupyNext(currentTime, acquireCount, count);
            if (waitInMs < OccupyTimeoutProperty.getOccupyTimeout()) {
                node.addWaitingRequest(currentTime + waitInMs, acquireCount);
                node.addOccupiedPass(acquireCount);
                sleep(waitInMs);

                // PriorityWaitException indicates that the request will pass after waiting for {@link @waitInMs}.
                throw new PriorityWaitException(waitInMs);
            }
        }
        return false;
    }
    return true;
}
```
此种策略比较简单粗暴，超过流量阈值的会直接拒绝。不过这里有一个小细节，如果入口流量prioritized为true，也就是优先级比较高，则会通过占用未来时间窗口的名额来实现。这个在上一篇文章有介绍到 []()

### 预热启动

`WarmUpController` 主要是用来防止流量的突然上升，使系统本在稳定状态下能处理的，但是由于许多资源没有预热，导致处理不了了。**注意这里的预热并不是指系统启动之后的一次性预热，而是指系统在运行的任何时候流量从低峰到突增的预热阶段**。

下面我们来看下`WarmUpController`的具体实现类

```java

/**
 * WarmUpController 构造方法
 * @param count 当前qps阈值
 * @param warmUpPeriodInSec 预热时长 秒
 * @param coldFactor 冷启动系数 默认为3
 */
private void construct(double count, int warmUpPeriodInSec, int coldFactor) {

    if (coldFactor <= 1) {
        throw new IllegalArgumentException("Cold factor should be larger than 1");
    }

    this.count = count;

    this.coldFactor = coldFactor;

    // 剩余Token的警戒值，小于警戒值系统就进入正常运行期
    warningToken = (int)(warmUpPeriodInSec * count) / (coldFactor - 1);
    // 系统最冷时候的剩余Token数
    maxToken = warningToken + (int)(2 * warmUpPeriodInSec * count / (1.0 + coldFactor));

    // 系统预热的速率(斜率)
    slope = (coldFactor - 1.0) / count / (maxToken - warningToken);

}

@Override
public boolean canPass(Node node, int acquireCount, boolean prioritized) {
    long passQps = (long) node.passQps();

    long previousQps = (long) node.previousPassQps();
    // 计算当前的 剩余 token 数
    syncToken(previousQps);

    // 如果进入了警戒线，开始调整他的qps
    long restToken = storedTokens.get();
    if (restToken >= warningToken) {
    // 计算剩余token超出警戒值的值
        long aboveToken = restToken - warningToken;
        // 计算当前允许通过的最大 qps
        double warningQps = Math.nextUp(1.0 / (aboveToken * slope + 1.0 / count));
        if (passQps + acquireCount <= warningQps) {
            return true;
        }
    } else {
    // 不在预热阶段，则直接判断当前qps是否大于阈值
        if (passQps + acquireCount <= count) {
            return true;
        }
    }

    return false;
}

```

首先是构造方法，主要关注2个重要参数

1. warningToken 剩余token的警戒值
2. maxToken  剩余的最大token数，如果剩余token数等于maxToken，则说明系统处于最冷阶段

要理解这两个参数的含义，可以参考令牌桶算法，每通过一个请求，就会从令牌桶中取走一个令牌。那么试想一下，当令牌桶中的令牌达到最大值是，是不是意味着系统目前处于最冷阶段，因为桶里的令牌始终处于一个非常饱和的状态。这里的令牌最大值对应的就是`maxToken`，而`warningToken`，则是对应了一个警戒值，当桶中的令牌数减少到一个指定的值时，说明系统已经度过了预热阶段

当一个请求进来时，首先需要计算当前桶中剩余的token数，具体逻辑在`syncToken`方法中
当系统剩余Token大于warningToken时，说明系统仍处于预热阶段，故需要调整当前所能通过的最大qps阈值

```java
protected void syncToken(long passQps) {
    long currentTime = TimeUtil.currentTimeMillis();
    // 获取秒级别时间（去除毫秒）
    currentTime = currentTime - currentTime % 1000;
    long oldLastFillTime = lastFilledTime.get();
    if (currentTime <= oldLastFillTime) {
        return;
    }

    long oldValue = storedTokens.get();
    // 判断是否需要往桶中添加令牌
    long newValue = coolDownTokens(currentTime, passQps);
    // 设置新的token数
    if (storedTokens.compareAndSet(oldValue, newValue)) {
    // 如果设置成功的话则减去上次通过的qps数量，就得到当前的实际token数
        long currentValue = storedTokens.addAndGet(0 - passQps);
        if (currentValue < 0) {
            storedTokens.set(0L);
        }
        lastFilledTime.set(currentTime);
    }

}

```

1. 获取当前时间
2. coolDownTokens 方法会判断是否需要往桶中放 token，并返回最新的token数
3. 如果返回了最新的token数，则将当前剩余的token数减去已经通过的qps，得到最新的剩余token数
 
```java
private long coolDownTokens(long currentTime, long passQps) {
    long oldValue = storedTokens.get();
    long newValue = oldValue;

    // 添加令牌的几种情况
    // 1. 系统初始启动阶段，oldvalue = 0，lastFilledTime也等于0，此时得到一个非常大的newValue，会取maxToken为当前token数量值
    // 2. 系统处于预热阶段 且 当前qps小于 count / coldFactor
    // 3. 系统处于完成预热阶段
    if (oldValue < warningToken) {
        newValue = (long)(oldValue + (currentTime - lastFilledTime.get()) * count / 1000);
    } else if (oldValue > warningToken) {
        if (passQps < (int)count / coldFactor) {
            newValue = (long)(oldValue + (currentTime - lastFilledTime.get()) * count / 1000);
        }
    }
    return Math.min(newValue, maxToken);
}
```
这里看一下会添加令牌的几种情况

1. 系统初始启动阶段，oldvalue = 0，lastFilledTime也等于0，此时得到一个非常大的newValue，会取maxToken为当前token数量值
2. 系统处于完成预热阶段，需要补充 token 使其稳定在一个范围内
3. 系统处于预热阶段 且 当前qps小于 count / coldFactor

前2种情况比较好理解，这里主要解释一下第三种情况，为何 当前`qps`小于`count / coldFactor`时，需要往桶中添加Token？试想一下如果没有这一步会怎么样，如果没有这一步在比较低的qps情况下补充Token，系统最终也会慢慢度过预热阶段，但实际上这么低的qps(`小于 count / coldFactor时`)不应该完成预热。所以这里才会在 qps低于`count / coldFactor`时补充剩余token数，**来让系统在低qps情况下始终处于预热状态下**


### 排队等待

排队等待的实现相对预热启动实现比较简单

首先会通过我们的配置，计算出相邻两个请求允许通过的最小时间，然后会记录最近一个通过的时间。两者相加即是下一次请求允许通过的最小时间。

```java
public boolean canPass(Node node, int acquireCount, boolean prioritized) {
    if (acquireCount <= 0) {
        return true;
    }
    if (count <= 0) {
        return false;
    }

    long currentTime = TimeUtil.currentTimeMillis();
    // 计算相隔两个请求 需要相隔多长时间
    long costTime = Math.round(1.0 * (acquireCount) / count * 1000);

    // 本次期望通过的最小时间
    long expectedTime = costTime + latestPassedTime.get();
    // 如果当前时间大于期望时间，说明qps还未超过阈值，直接通过
    if (expectedTime <= currentTime) {
       
        latestPassedTime.set(currentTime);
        return true;
    } else {
        // 当前时间小于于期望时间，请求过快了，需要排队等待指定时间
        
        // 计算等待时间
        long waitTime = costTime + latestPassedTime.get() - TimeUtil.currentTimeMillis();
        // 等待时长大于我们设置的最大时长，则不通过
        if (waitTime > maxQueueingTimeMs) {
            return false;
        } else {
        // 否则则排队等待，占用下通过时间
            long oldTime = latestPassedTime.addAndGet(costTime);
            try {
        
                waitTime = oldTime - TimeUtil.currentTimeMillis();
                // 判断等待时间是否已经大于最大值
                if (waitTime > maxQueueingTimeMs) {
                // 大于则将上一步加的值重新减去
                    latestPassedTime.addAndGet(-costTime);
                    return false;
                }
                // in race condition waitTime may <= 0
                // 占用等待时间成功，直接sleep costTime
                if (waitTime > 0) {
                    Thread.sleep(waitTime);
                }
                return true;
            } catch (InterruptedException e) {
            }
        }
    }
    return false;
}
```

排队等待控制器的核心策略其实就是围绕了`latestPassedTime`进行的，`latestPassedTime`指的是上一次请求通过的时间，通过`latestPassedTime` + `costTime`来与当前时间做比较，来判断当前请求是否可以通过，无法通过的请求则会优先占用`latestPassedTime`时间，直到sleep到可以通过的时间。当然我们也可以配置排队等待的最大时间，来限制目前排队等待通过的请求数量。

### 预热启动排队等待

预热排队等待，`WarmUpRateLimiterController`实现类我们发现其继承了`WarmUpController`，这是Sentinel在1.4版本后新加的一种控制器，其实就是预热启动和排队等待的结合体，具体源码我们就不做分析。

## 尾言

`Sentinel`的流控策略和流控效果的相结合使用还是非常巧妙的，当中的一些设计思想还是非常有借鉴意义的

Sentinel系列

* [Sentinel源码解析一（流程总览）](https://www.cnblogs.com/taromilk/p/11750962.html)

* [Sentinel源码解析二（slot总览）](https://www.cnblogs.com/taromilk/p/11751000.html)

* [Sentinel源码解析三（滑动窗口流量统计）](https://www.cnblogs.com/taromilk/p/11751009.html)
* [Sentinel源码解析四（流控策略和流控效果）](https://www.cnblogs.com/taromilk/p/11877242.html)



