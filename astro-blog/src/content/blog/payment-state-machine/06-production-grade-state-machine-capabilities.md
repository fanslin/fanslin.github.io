---
title: "06 从 Demo 到生产级状态机还缺什么"
description: "从审计日志、幂等处理、动作扩展、条件校验和异常恢复等角度，梳理生产级状态机相对 Demo 还缺的能力。"
pubDate: 2026-05-25
tags:
  - 支付系统
  - 状态机
  - Java
---

## 一个支付场景问题

当前 demo 已经能做到：

```text
当前状态 + 事件 -> 目标状态
```

也能拒绝非法流转。

但如果放进真实支付系统，还会马上遇到新问题：状态变化要不要记录日志？重复通知算成功还是失败？支付成功后要不要发消息？金额不匹配时能不能推进？这些都不是最小 demo 能解决的。

## 生产级状态机不只是 Map

当前 demo 的核心结构可以理解为：

```text
Map<StatusEventPair, Status>
```

生产级状态机通常需要升级成：

```text
Map<StatusEventPair, Transition>
```

也就是说，状态事件对不只映射到目标状态，还要映射到一次完整的转移动作。

## Transition 应该包含什么

一个更完整的 `Transition` 可以包含：

```text
sourceStatus
event
targetStatus
guard
action
idempotentPolicy
description
```

其中：

- `sourceStatus`：原状态。
- `event`：触发事件。
- `targetStatus`：目标状态。
- `guard`：推进前的条件判断。
- `action`：推进成功后的业务动作。
- `idempotentPolicy`：重复事件如何处理。
- `description`：用于日志和排障。

这样状态机就不只是查表，而是能承载更完整的业务语义。

## guard：推进前先判断条件

支付成功不只看事件，还可能要检查金额、币种、渠道流水号、商户号是否匹配。

伪代码可以是：

```java
STATE_MACHINE
    .accept(PAYING, PAY_SUCCESS, PAID)
    .guard(context -> context.amountMatched())
    .guard(context -> context.channelMatched());
```

如果 guard 不通过，即使当前状态和事件匹配，也不能推进到成功。

## action：推进成功后做什么

状态推进成功后，通常还要执行后续动作：

- 记录状态流转日志。
- 发送支付成功消息。
- 触发记账。
- 通知订单系统。
- 唤起后续补偿或对账任务。

伪代码：

```java
STATE_MACHINE
    .accept(PAYING, PAY_SUCCESS, PAID)
    .action(context -> context.appendTransitionLog())
    .action(context -> context.publishPaymentSuccessEvent());
```

action 要注意事务边界。状态更新和消息发送之间通常需要可靠消息、事务消息或 outbox 模式保证一致性。

## TransitionResult：别只返回成功或异常

最小 demo 用异常表达非法推进。生产系统里可以更细：

```text
SUCCESS
REJECTED
IDEMPOTENT
RETRYABLE
```

含义：

- `SUCCESS`：状态推进成功。
- `REJECTED`：当前状态不接受这个事件。
- `IDEMPOTENT`：重复事件，状态保持不变，但业务上视为已处理。
- `RETRYABLE`：临时失败，可以稍后重试。

有了结果类型，调用方就能更精确地决定是告警、忽略、重试，还是进入人工处理。

## 状态流转日志：支付系统必须能解释自己

每次状态推进都应该记录：

```text
payment_id
source_status
event
target_status
result
reason
request_id
message_id
created_at
```

日志的价值不只是排错。它还可以支持对账、审计、运营查询、问题复盘。

支付系统里，“为什么这笔单最终是成功”必须能解释清楚。

## 补偿机制：状态机不是万能药

状态机能保证本系统内部状态推进合法，但不能保证外部渠道永远可靠。

生产系统仍然需要：

- 定时查单。
- 消息重试。
- 超时关闭。
- 渠道对账。
- 人工差错处理。

状态机负责把每一次补偿结果纳入合法状态流转，补偿机制负责把外部不确定性最终收敛。

## 本篇小结

最小 demo 解决的是“能不能转”。生产级状态机还要解决“为什么能转、转完做什么、重复怎么办、失败如何解释、外部不确定性怎么兜底”。

下一篇会继续深入一个支付核心问题：并发、幂等和数据库一致性。
