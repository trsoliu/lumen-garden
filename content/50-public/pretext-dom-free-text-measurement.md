---
title: Pretext：AI 应用里的文本排版，不该每次都问 DOM
date: 2026-06-08
updated: 2026-06-08
verified: 2026-06-08
status: public-candidate
tags:
  - ai
  - frontend
  - performance
  - typography
  - pretext
publish: true
slug: pretext-dom-free-text-measurement
summary: Pretext 的价值不只是更快地算文本高度，而是把 AI 应用里的文本测量从容易触发回流的 DOM 读操作，拆成可缓存、可复用、可验证的布局步骤。
---

# Pretext：AI 应用里的文本排版，不该每次都问 DOM

大模型让很多应用重新变成“文本密集型系统”。

聊天界面会持续流式追加回答；agent 控制台会保留越来越长的历史；AI 生成 UI 时，按钮、卡片、表格和提示文案都可能在最后一刻才出现。过去可以交给浏览器慢慢排的文本，现在经常要在渲染前就知道它大概占多高、会不会换行、会不会把布局顶开。

这就是 Pretext 值得关注的地方。它不是一个用来替代 CSS 的完整排版引擎，而是一个更窄、更工程化的工具：在不依赖 DOM 测量的情况下，计算多行文本的高度和行布局。

本文由一篇本地 Pretext 初稿改写而来，并在 2026-06-08 对 Pretext 官方 README、package metadata 和已归档的官方研究记录做了核验。为了稳妥，本文不沿用本地初稿中尚未同日复核的具体 benchmark 数字、GitHub star 数和作者履历，只保留官方资料可以支撑的机制、API 和限制。

## 真正的问题不是文本，而是测量

Web 前端里，想知道一段文本渲染后有多高，是一个非常普通的需求。

虚拟列表要提前知道每一项的高度；聊天气泡要决定宽度和换行；瀑布流要估计卡片占位；编辑器、Canvas、SVG 或 WebGL 渲染也可能需要手动控制每一行文字的位置。

传统做法通常是把文本放进 DOM，然后读取 `getBoundingClientRect()`、`offsetHeight` 之类的测量结果。问题在于，这类 DOM 读操作可能迫使浏览器同步计算布局。一次两次没什么；但如果你在长列表、流式消息或布局验证循环里反复测量，就容易把性能成本放大。

更麻烦的是读写交错：改样式、读高度、再改样式、再读高度。浏览器为了回答这些问题，可能不得不一遍遍刷新布局状态。这类 forced layout 或 layout thrashing，常常不是出现在最显眼的动画里，而是藏在“我只是想知道这段字有多高”的工具函数里。

AI 应用会放大这个问题，因为文本不再只是静态内容。它会不断生成、不断变长、不断被重新包装进 UI 结构里。渲染之后再发现高度不对、按钮溢出、滚动锚点跳走，已经太晚了。

## Pretext 的切法：把重活和热路径分开

Pretext 官方 README 把它描述为一个纯 JavaScript/TypeScript 的多行文本测量和布局库。当前抓取到的 package metadata 显示包名是 `@chenglou/pretext`，版本为 `0.0.7`，许可证为 MIT。

它最重要的设计，是把工作拆成两个阶段：

```ts
import { prepare, layout } from '@chenglou/pretext'

const prepared = prepare(text, '16px Inter')
const { height, lineCount } = layout(prepared, 320, 24)
```

`prepare()` 做一次性工作：处理空白字符、分割文本、应用断行相关规则，并通过 Canvas 2D 的文本测量能力取得片段宽度。这个阶段和文本内容、字体配置有关。

`layout()` 则拿着已经准备好的结果，在给定最大宽度和行高后计算高度与行数。官方 README 把它定位成便宜的 hot path：当容器 resize 时，应该复用同一个 prepared handle，只重新跑 `layout()`，而不是反复重新 `prepare()`。

这个边界很关键。Pretext 不是说“文本测量没有成本”，而是把成本移到更可控的位置：

- 文本或字体变化时，做 `prepare()`
- 容器宽度变化时，做 `layout()`
- 避免为了每次宽度试探都去读 DOM 布局
- 让虚拟化、预估高度和开发期校验可以在浏览器布局之外先跑一轮

这种设计也解释了为什么它和 AI 界面有关：AI 输出的文本经常会在多个宽度、多个布局候选、多个组件变体之间被反复试探。热路径越便宜，agent 或渲染层越容易把“先验证再显示”变成常规流程。

## 它能解决哪些场景

第一个场景是虚拟滚动和长聊天历史。

虚拟列表最怕高度估错。高度估错会影响滚动位置、可见区计算和新消息进入时的锚点稳定。Pretext 的价值在于，文本高度可以在渲染前先按同一套字体和行高计算出来。它不必替代最终 DOM 渲染，但可以减少“先渲染隐藏节点再测量”的依赖。

第二个场景是 AI 生成 UI 的文案校验。

AI 生成的按钮文案、卡片标题、导航项或表格列名，很容易在英文、中文、阿拉伯文、emoji 或混合文本里产生意料之外的换行。官方 README 也把“开发期验证标签是否溢出”列为用法之一。对 design-to-code 或 UI agent 来说，这意味着可以在截图验收之前，先做一层更便宜的文本布局检查。

第三个场景是 Canvas、SVG、WebGL 这类手动渲染。

如果文本最终不是交给 DOM 排版，而是由应用自己逐行绘制，那么你需要的不只是总高度，还可能是每一行的范围、宽度和 cursor。Pretext 提供了 `prepareWithSegments()`、`layoutWithLines()`、`walkLineRanges()`、`measureLineStats()`、`layoutNextLineRange()` 等手动布局 API，用来支持固定宽度逐行布局、只统计行信息、或每一行使用不同宽度的布局。

第四个场景是布局位移预防。

当新文本异步加载时，如果应用能提前知道占位高度，就更容易避免内容突然下推，也更容易保持滚动锚点。对大模型应用来说，这尤其常见：回答还在流式增长，用户又可能正在阅读上文。

## 它不是什么

Pretext 不应该被理解成“完整复刻浏览器排版”。

官方 README 的 caveats 很重要：它依赖 `Intl.Segmenter` 和 Canvas 2D 文本测量；调用者需要让传给 Pretext 的 font、line-height、letter-spacing 等配置与实际 CSS 保持一致；它支持的是一组明确的文本布局选项，而不是全部 CSS 文本排版能力。

还有一个容易忽略的边界：官方文档明确提醒，`system-ui` 在 macOS 上可能因为字体解析差异影响 `layout()` 准确性。如果你需要严肃匹配实际渲染，应该使用明确的字体名，并把字体加载状态纳入验证。

所以更稳妥的选型判断是：

- 如果你只是渲染普通正文，浏览器 CSS 仍然是默认方案。
- 如果你需要在渲染前估算大量文本高度，Pretext 值得评估。
- 如果你需要在 resize、虚拟化或布局搜索中反复试探宽度，Pretext 的两阶段模型很合适。
- 如果你需要完全复刻复杂 CSS inline formatting、字体 fallback、浏览器差异和所有文本特性，Pretext 不是一个无条件答案。

## 对 AI 应用的启发

Pretext 最有意思的地方，不只是“文本测量更快”。更大的启发是：AI 生成的 UI 不应该只靠最终截图来发现问题。

截图当然重要，但截图是比较晚的验证。到了截图阶段，DOM 已经渲染，布局已经发生，错误也已经进入页面。如果某些问题能在更早阶段用程序化方式检查，例如：

- 这段按钮文案在 160px 内是否会换成两行
- 这个聊天气泡在当前断点下高度是多少
- 这组生成卡片的标题是否会导致瀑布流高度剧烈变化
- 这个 Canvas 标签在多语言文本下是否需要缩小或换行

那么 agent 就能在提交 UI 之前先筛掉一批明显失败的候选。

这也是我更愿意把 Pretext 归到“AI 时代的前端基础设施”，而不只是“一个文本测量库”的原因。LLM 让文本变成实时生成的界面材料；Pretext 这类工具则把文本布局的一部分变成可计算、可缓存、可验证的工程环节。

## 发布前复核清单

如果要把 Pretext 用在正式文章、选型文档或工程决策里，建议至少复核这些点：

- 重新确认 `@chenglou/pretext` 的最新版本、README 和 API caveats。
- 只引用自己能复现实验条件的 benchmark，不直接沿用旧稿数字。
- 区分“避免 DOM 测量触发 reflow”和“完全没有文本测量成本”。
- 确认目标运行环境具备 `Intl.Segmenter` 和 Canvas 2D text measurement。
- 用明确字体测试，不把 `system-ui` 当成精确匹配的前提。
- 如果要声称支持某个浏览器、Worker、Node、Deno 或 server-side 场景，发布前单独核验官方文档和实际构建。

Pretext 给出的不是一个夸张的万能解法，而是一条很清楚的工程路线：把文本布局从“每次问 DOM”改成“先准备，再快速计算”。在 AI 应用里，这条路线会越来越有用。

## 资料来源

核验时间：2026-06-08。

- [Pretext GitHub 仓库](https://github.com/chenglou/pretext)
- [Pretext README](https://raw.githubusercontent.com/chenglou/pretext/main/README.md)
- [Pretext package.json](https://raw.githubusercontent.com/chenglou/pretext/main/package.json)
- [Pretext RESEARCH.md](https://raw.githubusercontent.com/chenglou/pretext/main/RESEARCH.md)
