---
title: "08 成熟 Java 状态机方案怎么选"
description: "围绕业务复杂度、审计能力、团队维护成本和支付链路稳定性，比较手写状态机与成熟 Java 状态机框架的适用边界。"
pubDate: 2026-05-25
tags:
  - 支付系统
  - 状态机
  - Java
---

## 一个支付场景问题

手写状态机已经能工作了，那还要不要引入框架？

这个问题不能只看“框架功能多不多”。支付核心链路更关心：规则是否清楚、行为是否可审计、异常是否可恢复、团队是否能长期维护。

所以状态机选型要从业务复杂度出发。

## 方案一：当前实现增强版

如果支付状态机规模不大，可以在当前 demo 上增强。

适合场景：

- 状态和事件数量有限。
- 团队希望少依赖外部框架。
- 对规则透明度要求高。
- 状态机主要服务核心交易链路。

建议增强方向：

```text
Map<StatusEventPair, Transition>
```

让 `Transition` 包含：

```text
sourceStatus
event
targetStatus
guard
action
idempotentPolicy
description
```

优点是简单、透明、可控。缺点是高级能力要自己补。

## 方案二：COLA StateMachine

COLA 是阿里开源的应用架构实践，其中包含 `cola-component-statemachine`。

它适合 DDD/COLA 风格项目，也适合希望用轻量 fluent API 表达状态机的业务系统。

典型表达类似：

```java
builder.externalTransition()
    .from(PaymentStatus.PAYING)
    .to(PaymentStatus.PAID)
    .on(PaymentEvent.PAY_SUCCESS)
    .when(checkCondition())
    .perform(doAction());
```

优势：

- 比手写 `Map` 更表达式化。
- 支持 condition 和 action。
- 和领域建模结合自然。
- 对业务开发比较友好。

注意点：

- 如果项目不是 COLA 架构，也可以只借鉴状态机组件。
- 引入前要确认团队是否接受它的工程组织方式。

参考：[GitHub - alibaba/COLA](https://github.com/alibaba/COLA)

## 方案三：Spring Statemachine

Spring Statemachine 是 Spring 官方状态机项目，能力比较完整。

适合场景：

- 项目已经深度使用 Spring。
- 状态层级复杂。
- 需要 guard、action、listener、持久化配置。
- 需要 entry/exit action 或状态机测试工具。

简化后的配置感觉类似：

```java
transitions
    .withExternal()
    .source(PAYING)
    .target(PAID)
    .event(PAY_SUCCESS)
    .guard(paymentMatchedGuard)
    .action(paymentSuccessAction);
```

优势是功能完整、生态成熟。缺点是对简单支付状态机来说可能偏重。

支付核心状态最终仍应落数据库，并配合事务、锁、版本号和补偿机制。不要因为用了框架，就误以为一致性问题自动解决了。

参考：[Spring Statemachine](https://spring.io/projects/spring-statemachine/)

## 方案四：Squirrel

Squirrel Foundation 是一个 Java 状态机库，强调 lightweight、type safe、programmable、diagnosable。

适合场景：

- 想要比手写状态机更完整的抽象。
- 不想引入 Spring Statemachine 那么重的框架。
- 希望有类型安全和诊断能力。

优势是轻量、可编程、比自研少踩一些通用坑。

注意点是需要评估维护活跃度、版本兼容性和团队熟悉程度。

参考：[GitHub - hekailiang/squirrel](https://github.com/hekailiang/squirrel)

## 方案五：Apache Commons SCXML

Apache Commons SCXML 是 Java SCXML 引擎。SCXML 用 XML 描述状态机，适合状态机外部化和标准化。

适合场景：

- 状态机需要配置化。
- 业务方或平台需要审阅状态图和状态定义。
- 状态机不仅服务支付，还服务工作流、页面流、交互流程。

优势是标准化表达能力更强。缺点是 XML 对普通业务开发不一定友好，对支付核心链路可能显得重。

参考：[Apache Commons SCXML](https://commons.apache.org/proper/commons-scxml/)

## 方案六：stateless4j / StatefulJ

这类方案适合想要简单 fluent API，又不想引入完整大型框架的场景。

适合场景：

- 中小型状态机。
- 希望代码比手写 `Map` 更顺手。
- 不需要复杂层级状态或平台化配置。

参考：

- [Maven Central - stateless4j](https://central.sonatype.com/artifact/com.github.stateless4j/stateless4j)
- [StatefulJ Introduction](https://www.statefulj.io/)

## 怎么选

| 场景 | 推荐方案 |
| --- | --- |
| 学习状态机、验证核心思想 | 当前 demo |
| 支付主单状态较少，团队重视透明和可控 | 当前实现增强版 |
| DDD/COLA 架构项目 | COLA StateMachine |
| Spring 项目且状态层级复杂 | Spring Statemachine |
| 需要轻量、类型安全、可扩展库 | Squirrel |
| 状态机要配置化、标准化、平台化 | Apache Commons SCXML |
| 只是想要简单 fluent API | stateless4j / StatefulJ |

## 我的建议

支付核心链路不要为了框架而框架。

如果状态机规模不大，当前实现增强版已经足够好。它透明、可控、容易审计。

如果项目本身就是 DDD/COLA 风格，可以优先看 COLA StateMachine。

如果状态机非常复杂，并且项目深度依赖 Spring，再考虑 Spring Statemachine。

不管选哪种方案，都不要忘记：支付状态机的核心不是 API 多漂亮，而是状态清楚、事件合法、终态不可回退、并发可控、结果可追踪。

## 本篇小结

成熟框架能减少样板代码，也能提供 guard、action、listener、持久化等能力。但支付系统最终拼的是工程确定性。

状态机框架只是表达规则的工具。真正让支付系统稳定的，是清晰的生命周期设计、严格的状态推进规则、数据库一致性、幂等策略、流转日志和补偿机制。
