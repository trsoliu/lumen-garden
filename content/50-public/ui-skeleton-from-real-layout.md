---
title: 骨架屏应该从真实 UI 派生
date: 2026-06-09
updated: 2026-06-09
status: public-candidate
tags:
  - frontend
  - performance
  - ui
  - ai
publish: true
slug: ui-skeleton-from-real-layout
summary: 从 Boneyard、npm 和 MUI Skeleton 文档出发，解释为什么 AI 辅助改版会放大手工骨架屏的同步成本，以及把 skeleton 变成真实 UI 派生产物的边界。
verified: 2026-06-09
---

# 骨架屏应该从真实 UI 派生

骨架屏没有过时。真正开始过时的是把骨架屏当成另一份手工布局来长期维护。

在稳定页面里，手写 skeleton 完全合理。MUI 这类成熟组件库提供的 `Skeleton`，就是为了在数据加载前给用户一个内容占位预览，减少等待时的挫败感。问题出现在另一种场景：页面本身开始被 AI、设计系统和人工评审一起高频改动。

当真实 UI 一周改三次，loading UI 却还是上个月手写的副本，失真会很快出现：

- 真实卡片多了 badge、摘要或按钮，骨架屏没有同步。
- 移动端已经改单列，loading 态还保持旧的双列结构。
- 数据回来后页面明显跳动，用户看到的是两个不一致的页面。
- PR 和 QA 都盯着最终态，loading state 变成最容易漏掉的部分。

所以 AI 时代的骨架屏问题，不是“要不要画灰块”，而是“为什么我们还默认维护两份布局”。

## Boneyard 的关键变化

`boneyard-js` 的 README 把定位说得很清楚：它要从真实 UI 中提取 skeleton，而不是让开发者重新手工测量一套占位布局。它支持 React、Preact、Vue、Svelte 5、Angular 和 React Native；Web 端的流程是由 CLI 或 Vite 插件打开浏览器，访问应用，找到 `<Skeleton name="...">`，在多个断点下 snapshot 布局，然后输出静态 bones 数据。

这件事真正重要的不是“自动生成骨架屏”，而是来源关系反过来了。

传统方式通常是：

1. 写真实页面。
2. 另写一份 `UserCardSkeleton`。
3. 以后靠工程师记得同步两份布局。

Boneyard 的方式更接近：

1. 维护真实页面。
2. 用 `<Skeleton>` 声明加载态边界。
3. 在构建时从真实渲染结果派生骨架数据。
4. 运行时按当前容器或断点选择合适的 bones。

也就是说，skeleton 从“手写组件层”下沉成了“构建产物层”。这比换一种 shimmer 动画更有价值。

## AI 在这里不应该手写 skeleton

AI 当然可以帮你改 UI，但它不应该被要求再手写一份近似的 loading 组件。那会把团队重新带回“双份布局”的老问题。

更合理的协作方式是：

```text
请只修改真实组件和用于构建时捕获布局的 fixture。
保留 <Skeleton name="user-card" loading={isLoading}> 边界。
不要额外手写 UserCardSkeleton。
改完后运行 boneyard-js build 重新生成 bones。
```

这条分工很关键：AI 负责加速真实 UI 的演化，构建流程负责把真实 UI 折叠成新的 skeleton 数据。这样加载态跟随事实来源变化，而不是依赖人类记忆同步。

## 什么时候值得用

自动骨架屏最适合三类页面：

- 信息结构稳定、字段和布局常变的后台页面。
- 列表、表单、详情页、数据卡片这些 DOM 结构明确的业务界面。
- 桌面、平板、移动端布局差异明显，手写一份通用 skeleton 很容易失真的页面。

它不一定适合所有地方。营销首页、强艺术化页面、Canvas 或地图主导页面、虚拟滚动极重的界面，都可能需要单独判断。

真正落地时，也不应该从全站替换开始。更稳的路径是选一个高频变化的结构化页面做 PoC，然后观察三件事：

- loading 态和真实态的视觉一致性是否提升。
- 布局跳动、断点错位、旧 skeleton 残留是否减少。
- CI 或本地构建是否能可靠地更新 bones，而不是制造新的维护负担。

## 边界

这不是说手写 skeleton 错了。稳定页面、小组件、低风险加载态，手写仍然简单可靠。

Boneyard 这类思路更适合解决的是另一类问题：当 UI 的变化速度被 AI 和组件化流程抬高以后，loading UI 也需要从“副本资产”变成“派生产物”。否则页面越快进化，骨架屏越快变旧。

未来前端团队需要维护的可能不是更多 skeleton 组件，而是更好的事实来源：真实组件、代表性 fixture、构建命令、断点策略和视觉回归检查。

References:

- [Boneyard GitHub README](https://github.com/0xGF/boneyard)
- [boneyard-js on npm](https://www.npmjs.com/package/boneyard-js)
- [MUI Skeleton documentation](https://mui.com/material-ui/react-skeleton/)
