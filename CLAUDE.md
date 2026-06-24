# CLAUDE.md

## 项目概述

media-tools 是一个 Claude Code Skill，用于一键生成前端页面的截图和展示视频。

阅读 `CONTEXT.md` 获取完整的设计上下文和决策记录。

## 参考文件

- `references/screenshot-example.mjs` — 截图脚本参考实现
- `references/composition-example.html` — HyperFrames composition 参考实现

## 目标

创建一个可分发的 Claude Code Skill（npm 包），用户安装后可以通过 `/media-tools` 命令一键生成页面展示视频。

## 技术栈

- Puppeteer — 截图
- HyperFrames — HTML/GSAP → MP4 视频合成
- Claude Code Skill — 分发和使用形式

## 当前状态

项目刚初始化，准备开始实现。阅读 `CONTEXT.md` 了解完整设计。
