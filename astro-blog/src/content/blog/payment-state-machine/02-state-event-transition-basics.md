---
title: "02 状态、事件、转移规则三件套"
description: "拆解状态机的三个核心概念：状态、事件和转移规则，并用支付主单流转示例说明它们如何协作。"
pubDate: 2026-05-25
tags:
  - 支付系统
  - 状态机
  - Java
---

## 一个支付场景问题

如果只说“支付单状态从 `PAYING` 变成 `PAID`”，这句话其实少了一个关键信息：为什么会变？

是渠道异步通知成功？是定时查单查到成功？是人工补单？还是测试代码直接改了字段？

状态机要求我们把“状态”和“触发状态变化的事件”分开看。

## 状态：对象当前处于什么阶段

当前 Java demo 里，支付状态定义在 `PaymentStatus`：

```java
public enum PaymentStatus implements BaseStatus {
    INIT("初始化"),
    PAYING("支付中"),
    PAID("支付成功"),
    FAILED("支付失败");
}
```

这里的状态只描述支付主单自己的生命周期：

```text
INIT -> PAYING -> PAID/FAILED
```

注意，退款状态没有放进这里。支付主单到 `PAID` 就是支付成功，后续退款应该交给退款单处理。

## 事件：业务上发生了什么

支付事件定义在 `PaymentEvent`：

```java
public enum PaymentEvent implements BaseEvent {
    PAY_CREATE("支付创建"),
    PAY_PROCESS("支付中"),
    PAY_SUCCESS("支付成功"),
    PAY_FAIL("支付失败");
}
```

事件不是状态。`PAY_SUCCESS` 表示“支付成功事件发生了”，而 `PAID` 表示“支付单当前已经处于支付成功状态”。

这层区分很重要。业务代码应该发事件，而不是直接指定目标状态。

## 转移规则：当前状态收到事件后能去哪里

当前 demo 把合法转移集中定义在 `PaymentStatus` 的静态块：

```java
STATE_MACHINE.accept(null, PaymentEvent.PAY_CREATE, INIT);
STATE_MACHINE.accept(INIT, PaymentEvent.PAY_PROCESS, PAYING);
STATE_MACHINE.accept(PAYING, PaymentEvent.PAY_SUCCESS, PAID);
STATE_MACHINE.accept(PAYING, PaymentEvent.PAY_FAIL, FAILED);
```

它可以读成一张表：

| 当前状态 | 事件 | 目标状态 |
| --- | --- | --- |
| `null` | `PAY_CREATE` | `INIT` |
| `INIT` | `PAY_PROCESS` | `PAYING` |
| `PAYING` | `PAY_SUCCESS` | `PAID` |
| `PAYING` | `PAY_FAIL` | `FAILED` |

没有出现在表里的组合，就是不允许。

## 为什么事件比 setStatus 更可靠

如果业务代码直接调用：

```java
paymentModel.setStatus(PaymentStatus.PAID);
```

它绕过了状态机。任何地方都可以直接把状态改成成功。

而通过事件推进：

```java
paymentModel.transferStatusByEvent(PaymentEvent.PAY_SUCCESS);
```

状态机必须先检查当前状态。如果当前不是 `PAYING`，这次推进就会失败。

## 最小规则也要显式定义

哪怕只有 4 个状态、4 个事件，也建议显式写转移表。因为支付状态变化不是普通赋值，它是交易事实变化。

状态机最基础的设计原则可以压缩成四句话：

- 状态要少而明确。
- 事件要表达业务事实。
- 转移规则要集中。
- 未定义的转移默认拒绝。

## 本篇小结

状态机的三件套是：状态、事件、转移规则。

状态回答“现在是什么”，事件回答“发生了什么”，转移规则回答“当前状态收到这个事件后能不能变，以及变成什么”。

下一篇会用 Java 手写一个最小可运行状态机，把这张转移表变成代码。
