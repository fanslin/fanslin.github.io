---
title: "支付状态机技术演进：系列目录"
description: "按从简到难的路线梳理支付状态机系列文章，覆盖状态流转、Java 实现、生产级能力、并发一致性和框架选型。"
pubDate: 2026-05-25
tags:
  - 支付系统
  - 状态机
  - Java
---

这个系列把支付状态机按“从简到难”的路线拆开讲：先从支付系统里的真实状态问题出发，再到最小 Java 实现，最后扩展到生产级能力、并发一致性和成熟框架选型。

适合读者：

- 写过订单、支付、退款状态字段，但还没有系统设计过状态机的开发者。
- 用过 `if/else` 或 `switch` 推进状态，开始感觉维护困难的后端开发者。
- 想把支付系统里的状态流转、幂等、并发和补偿机制串起来理解的工程师。

## 推荐阅读顺序

1. [为什么支付系统需要状态机](/blog/payment-state-machine/01-why-payment-needs-state-machine/)
2. [状态、事件、转移规则三件套](/blog/payment-state-machine/02-state-event-transition-basics/)
3. [手写一个最小可运行 Java 状态机](/blog/payment-state-machine/03-minimal-java-state-machine-implementation/)
4. [终态防回退与乱序消息处理](/blog/payment-state-machine/04-terminal-state-and-out-of-order-events/)
5. [为什么不要把退款、请款、拒付都塞进支付主单](/blog/payment-state-machine/05-split-payment-refund-and-other-order-states/)
6. [从 Demo 到生产级状态机还缺什么](/blog/payment-state-machine/06-production-grade-state-machine-capabilities/)
7. [并发、幂等与数据库一致性](/blog/payment-state-machine/07-concurrency-idempotency-and-database-consistency/)
8. [成熟 Java 状态机方案怎么选](/blog/payment-state-machine/08-state-machine-frameworks-for-java/)

## 阅读路径建议

如果你只想先理解状态机的价值，读第 1、2 篇即可。

如果你想自己写一个能跑的 Java 版本，读第 2、3、4 篇。

如果你正在做支付、退款、请款等交易系统设计，重点读第 4、5、6、7 篇。

如果你正在做技术选型，直接读第 8 篇，但建议先看第 6、7 篇，否则容易只比较框架功能，而忽略支付核心链路真正需要的稳定性能力。

## 系列主线

状态机不是为了引入一个设计模式，而是为了回答支付系统里最危险的几个问题：

- 当前状态能不能接收这个事件？
- 旧消息会不会把终态改回中间态？
- 重复通知应该报错、忽略，还是幂等成功？
- 并发场景下，状态机判断成功后，数据库是否真的只更新了一次？
- 退款、请款、拒付这些生命周期，应该放在支付主单里，还是拆成独立单据？

这一组文章会围绕这些问题逐层展开。
