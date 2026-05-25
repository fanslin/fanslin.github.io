---
title: "03 手写一个最小可运行 Java 状态机"
description: "用 Java 从状态、事件、转移表和状态推进方法开始，实现一个最小可运行的支付状态机模型。"
pubDate: 2026-05-25
tags:
  - 支付系统
  - 状态机
  - Java
---

## 一个支付场景问题

状态机听起来像一个框架，但最小实现其实很简单。我们只需要能表达一件事：

```text
当前状态 + 事件 -> 目标状态
```

这一篇用当前 demo 展示如何用 Java 写出这个最小模型。

## 第一步：定义状态和事件的标记接口

状态和事件都可以先抽象成标记接口：

```java
public interface BaseStatus {
}
```

```java
public interface BaseEvent {
}
```

它们本身没有方法，主要用于泛型约束，让 `StateMachine` 可以服务不同业务，比如支付单、退款单、请款单。

## 第二步：定义状态事件对

状态机的 key 是“当前状态 + 事件”，所以需要一个 `StatusEventPair`：

```java
public final class StatusEventPair<S extends BaseStatus, E extends BaseEvent> {
    private final S status;
    private final E event;

    public StatusEventPair(S status, E event) {
        this.status = status;
        this.event = event;
    }
}
```

实际代码里还实现了 `equals` 和 `hashCode`，这样它才能作为 `Map` 的 key。

## 第三步：定义通用状态机

最小状态机就是一个映射表：

```java
public final class StateMachine<S extends BaseStatus, E extends BaseEvent> {
    private final Map<StatusEventPair<S, E>, S> statusEventMap = new HashMap<>();

    public void accept(S sourceStatus, E event, S targetStatus) {
        statusEventMap.put(new StatusEventPair<>(sourceStatus, event), targetStatus);
    }

    public S getTargetStatus(S sourceStatus, E event) {
        return statusEventMap.get(new StatusEventPair<>(sourceStatus, event));
    }
}
```

`accept` 用来注册合法转移，`getTargetStatus` 用来查询当前状态收到事件后能不能转移。

## 第四步：定义支付状态机

支付状态机规则集中写在 `PaymentStatus`：

```java
private static final StateMachine<PaymentStatus, PaymentEvent> STATE_MACHINE = new StateMachine<>();

static {
    STATE_MACHINE.accept(null, PaymentEvent.PAY_CREATE, INIT);
    STATE_MACHINE.accept(INIT, PaymentEvent.PAY_PROCESS, PAYING);
    STATE_MACHINE.accept(PAYING, PaymentEvent.PAY_SUCCESS, PAID);
    STATE_MACHINE.accept(PAYING, PaymentEvent.PAY_FAIL, FAILED);
}
```

这样规则非常直观：哪些状态可以接收哪些事件，一眼能看清。

## 第五步：领域模型只允许通过事件推进

`PaymentModel` 不暴露 `setStatus`，只暴露：

```java
public void transferStatusByEvent(PaymentEvent event) {
    PaymentStatus targetStatus = PaymentStatus.getTargetStatus(currentStatus, event);
    if (targetStatus == null) {
        throw new StateMachineException(currentStatus, event, "状态转换失败");
    }
    lastStatus = currentStatus;
    currentStatus = targetStatus;
}
```

这段代码是整个 demo 的核心。

如果当前状态和事件找不到目标状态，就抛出 `StateMachineException`。业务代码不能绕过状态机直接改状态。

## 为什么不用 if/else 或 switch

`if/else` 和 `switch` 在状态少的时候也能工作，但随着状态和事件增加，它们会变成分支网：

```java
if (status == INIT && event == PAY_PROCESS) {
    status = PAYING;
} else if (status == PAYING && event == PAY_SUCCESS) {
    status = PAID;
}
```

这种写法的问题是：规则被代码分支包住了，不再像表格一样清楚。后续新增状态时，很容易漏掉某些组合。

状态机把规则变成表，未定义的组合默认失败。

## 运行 demo

当前项目可以直接编译运行：

```bash
javac -d payment-state-machine-java/out $(find payment-state-machine-java/src -name '*.java')
java -cp payment-state-machine-java/out com.example.statemachine.PaymentStateMachineDemo
```

你会看到三段结果：

- 正常流转：`INIT -> PAYING -> PAID`
- 非法流转：`INIT` 不能直接 `PAY_SUCCESS`
- 终态防回退：`PAID` 不能回到 `PAYING`

## 本篇小结

最小状态机并不复杂：一个状态事件对、一个映射表、一个通过事件推进状态的方法，就能把状态规则集中起来。

下一篇会重点看支付系统里最容易出问题的部分：终态防回退和乱序消息。
