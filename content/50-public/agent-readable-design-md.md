---
title: DESIGN.md 是给 Agent 读的设计系统上下文
date: 2026-06-09
updated: 2026-06-09
status: public-candidate
tags:
  - ai
  - design-system
  - agent
  - frontend
publish: true
slug: agent-readable-design-md
summary: 基于 Google Stitch 的 DESIGN.md 公告和 google-labs-code/design.md 仓库，解释为什么 AI 时代设计系统需要一层 agent 可读的语义规则。
verified: 2026-06-09
---

# DESIGN.md 是给 Agent 读的设计系统上下文

如果只把 `DESIGN.md` 理解成“把设计规范写进 Markdown”，它看起来不会特别新。但 Google Labs 在 Stitch 里推动这件事，真正重要的地方不是文件后缀，而是协作对象变了。

过去设计系统的主要消费者是设计师和前端工程师。AI coding agent 加入真实交付之后，设计系统多了第三类消费者：会生成页面、改组件、重组信息层级的 agent。

这时问题就不再只是“设计稿在哪里”，而是：

> agent 能不能稳定继承一个产品的设计意图？

## 它补的是语义规则层

一个成熟产品通常已经有几类设计资产：

- Figma 或其他设计稿，用来表达页面、状态和原型。
- design tokens，用来标准化颜色、字号、间距、圆角等变量。
- 组件库，用来把规则落成可复用实现。

这些资产都重要，但它们并不天然告诉 agent 一件事：这些设计选择为什么这样组合，哪些地方必须克制，哪些地方允许变化。

比如一个 B2B 产品已经有统一的蓝色、灰阶、圆角和按钮组件。agent 用这些资产生成 Dashboard、账单页和项目列表，单页可能都没错，但放在一起仍然会漂移：一个页面留白很松，一个页面密度很高，一个页面的主按钮又过度抢眼。参数都合法，整体却不像同一个产品。

`DESIGN.md` 要补的就是这层语义规则：

- 产品整体应该给人什么感觉。
- 颜色分别承担什么角色，哪些能大面积使用，哪些只能点到为止。
- 字体层级如何表达信息优先级。
- 按钮、卡片、输入框、导航在界面里应该有多强的存在感。
- 哪些设计动作明确不该做。

这不是替代 Figma、tokens 或组件库，而是在它们之上加一层 agent 可读的解释层。

## Google 为什么把它开源

Google 在 2026 年 4 月 21 日发布的 Stitch 文章里说，`DESIGN.md` 可以让设计规则在项目之间导入导出，让 Stitch 理解设计系统背后的理由，并生成更贴合品牌的 UI。同一篇公告还把它描述成一个开源的草案规格，目标是让它不局限于单一工具或平台。

Google 的 `google-labs-code/design.md` 仓库也给出了更工程化的定位：这是一个描述视觉身份、给 coding agents 使用的格式规格，目标是让 agent 获得持久、结构化的设计系统理解。

这两点合在一起，信号很清楚：设计系统不只要给人看，也要能进入 agent 的上下文窗口，影响下一次生成。

## 好的 DESIGN.md 不只是参数表

如果一份 `DESIGN.md` 只列出品牌色、字号和阴影值，它会很快退化成另一份 token 文档。对 agent 真正有用的，是“参数 + 组合规则 + 禁区”。

比如按钮规则不能只写：

```md
Primary Button: #5B5BD6, radius 8px
```

更有用的写法接近：

```md
Primary Button
- Role: 只用于页面主行动，同一屏不要出现过多主按钮。
- Presence: 明确但克制，不使用营销页式发光效果。
- Shape: 中等圆角，避免过度胶囊化。
- Avoid: 不要同时把品牌强调色用于按钮背景、通知条和数据高亮。
```

第一种写法告诉 agent “值是什么”。第二种写法告诉 agent “这个组件在产品权力结构里应该是什么位置”。

AI 生成最容易出问题的，往往不是拿不到参数，而是拿不到边界。缺少边界时，agent 会把局部正确的设计动作叠在一起，最后得到一个整体不对的界面。

## 设计工作的重心会上移

当 AI 可以快速生成页面草图、布局变体和前端代码时，页面初稿会变便宜，一致性会变贵。

这会把设计工作的价值往上推：

- 从“画出一个页面”转向“定义可复用的设计规则”。
- 从“给人看的静态规范”转向“人和 agent 都能消费的上下文”。
- 从一次性 handoff 转向持续共享上下文。
- 从局部页面质量转向跨页面一致性治理。

强设计师不会因为 agent 能生成 UI 就失去价值。相反，设计师更像系统作者、生成策略制定者和审美 QA。他们要把品牌语言、组件边界和视觉判断写成可复用规则，再持续审阅 agent 输出是否偏航。

## 风险也要写进方法里

`DESIGN.md` 是一个方向，不是银弹。

第一，文本不等于标准。开源草案能降低协作门槛，但不同团队写出的质量会非常不均匀。

第二，抽象词很容易制造假清晰。比如“高级”“克制”“未来感”这些词，对人类似乎有感觉，但如果没有例子和禁区，agent 仍然会猜。

第三，公开案例适合学习，不等于可以复制品牌身份。团队应该提炼自己的视觉语言，而不是把别人的风格文件直接当模板套用。

第四，复杂交互、状态机、动画节奏和任务流程，很难完全压缩进静态文本。它们仍然需要原型、代码、测试和人工评审。

更稳的判断是：`DESIGN.md` 会成为 AI 时代设计系统新增的一层，而不是唯一的一层。它的价值在于把过去靠默契传递的设计意图，变成可以版本化、评审、复用，并能被 agent 稳定读取的上下文。

References:

- [Google Blog: Stitch's DESIGN.md format is now open-source](https://blog.google/innovation-and-ai/models-and-research/google-labs/stitch-design-md/)
- [google-labs-code/design.md](https://github.com/google-labs-code/design.md)
- [VoltAgent/awesome-design-md](https://github.com/VoltAgent/awesome-design-md)
