---
layout: post
title: "从一个 VSCode 插件开始：我对认识需求的一点反思"
date: 2026-05-05 00:00:00 +0800
categories: reflection
tags: [vscode, claude-code]
---

平时用第三方模型或者中转服务来使用 Claude Code，所以专门备了几个脚本文件，通过 `source` 来执行对应的脚本，设置不同提供商的 URL 和 token 等其它环境变量。

最近有在 VSCode 里面用 Claude Code 插件。相比终端，GUI 交互是会更方便直观一些。这个插件支持在 VSCode 的 `settings.json` 里配置特定的字段，然后根据这些字段接入第三方模型。

但问题来了，每次都要在 VSCode 里直接编辑 `settings.json` 来切换 provider，太繁琐了。于是突发奇想：是不是可以写一个插件来自动修改 `settings.json`？于是花了一点时间 vibe 了一个插件。

个人觉得这个插件还是有一点点亮点的。比如它在首次运行的时候，如果你之前在 VSCode 的 `settings.json` 里配置过模型接入相关字段，它能解析出来并自动生成一个默认配置。此外，这个插件还支持从 bash 脚本片解析出配置。

初步搞完这个插件之后，把它分享到了 V2EX 上。收到的评论大概四五条，普遍认为这个插件的价值不高——这点确实没法否认。然后发现，大家使用 Claude Code 接入第三方模型的方式和我都不太一样。也有人提到 CC-Switch 这类已有工具，言下之意是这个 VSCode 插件其实意义不大。

于是在 v2ex 的帖子下简单解释了一下我的这个插件和 CC-Switch 的差异。比如说，我的插件不会有系统级别的影响，只在 VSCode 里面生效，从某种程度上来说会比较"干净"一些。如果你同时在终端和 VSCode 里使用 Claude Code，两者可以使用不同的 provider，因为它们的配置方式是独立的——这也算是一个小小的优点，虽然可能只是我个人比较小众的需求。

不过，V2EX 上这些热心网友的反馈也给了我一些反思的空间。确实，市面上已经存在一些 VSCode 插件或工具能够起到切换 Claude Code provider 的功能，虽然实现方式和我的不太一样，但对大部分人来说，这确实不是大家所在意的痛点。

现在在想，是不是可以继续做一个专门管理 `settings.json` 的插件，切换 Claude Code 可以只是这个插件的某个具体的使用场景 （有一个麻烦的点是，插件能够在 `settings.json` 修改的字段是需要提前申明）。这样的一个插件可能更有需求空间。

以上大概就是我这两天的一些心路历程，也是对 AI 辅助编程的一次探索，以及对于"如何发现需求、确定需求"的一点认识。

最后感谢 v2ex 的热心网友，促成了后续的一些思考。

PS: 以上内容为语音转录后， 人工 + deepseek-v4-pro 整理所得

---

相关讨论：[https://www.v2ex.com/t/1210275](https://www.v2ex.com/t/1210275)

插件仓库：[https://github.com/jiang1997/claude-code-profile-manager](https://github.com/jiang1997/claude-code-profile-manager)
