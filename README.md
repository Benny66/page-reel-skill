# page-reel-skill

Claude Code Skill — 一键为前端项目生成页面展示视频。

安装后输入 `/media-tools`，AI 自动扫描项目路由、截图、生成带动画的 MP4 视频。

## 安装

```bash
npx skills add bennywen/page-reel-skill
```

## 使用

```
/media-tools
```

AI 会自动执行以下流程：

1. 扫描项目路由、守卫、API 接口
2. 识别功能模块，交互选择要展示的页面
3. 生成 Puppeteer 截图脚本 + HyperFrames 视频合成文件
4. 执行截图 → 校验 → 渲染 → 输出 MP4

## 支持的项目

- Vue 3 + Vite + vue-router（MVP）
- 支持 History / Hash 路由模式
- 自动检测后端服务，通过登录接口获取真实数据

## 配置

所有配置均可选，AI 会自动推断缺失项。

```yaml
# video-config.yaml
title: "我的项目展示"
subtitle: "页面功能展示"
viewport:
  width: 1440
  height: 900
modules:
  - dashboard
  - user-management
```

## 生成的文件

```
media/
├── screenshot.mjs          # 截图脚本（可重新运行）
├── index.html              # HyperFrames 视频合成文件
├── package.json            # HyperFrames 运行配置
├── hyperframes.json        # HyperFrames 路径配置
├── screenshots/            # 页面截图备份
├── assets/                 # 视频引用资源
│   └── fonts/              # 本地化字体文件
└── renders/                # 输出的 MP4 视频
```

## 前置依赖

| 依赖 | 用途 | 安装 |
|------|------|------|
| Puppeteer | 无头浏览器截图 | `npm install puppeteer` |
| HyperFrames | HTML/GSAP → MP4 视频合成 | `npx hyperframes@latest` |

## 重新生成

```bash
# 重新截图所有页面
node media/screenshot.mjs

# 重新截图指定页面
node media/screenshot.mjs --only=dashboard,manage

# 重新渲染视频
cd media && npm run render
```

## License

[MIT](LICENSE)
