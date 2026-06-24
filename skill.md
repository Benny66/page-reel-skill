# Media Tools

一键为前端项目生成页面展示视频。

## 触发条件

当用户输入 `/media-tools` 时激活此 Skill。

## 完整工作流程

执行以下步骤，每完成一步后向用户确认再继续下一步。

### Step 0: 读取配置

检查项目根目录是否存在 `video-config.yaml`：

```yaml
# video-config.yaml（所有字段均可选，AI 会自动推断缺失项）
title: "项目展示"           # 视频标题，不写则从 package.json 或目录名推断
subtitle: "页面功能展示"     # 副标题
# baseUrl: "http://localhost:5173"  # 不写则从 vite.config.ts 读取
# viewport:                         # 不写则默认 1440x900
#   width: 1440
#   height: 900
modules:                    # 不写则进入交互选择流程
  - dashboard
  - content-management
# pages:                    # 可选，手动指定页面（覆盖 modules）
#   - name: input
#     path: "#/input"
```

如果配置文件存在，读取所有字段。缺失的字段将在后续步骤中自动推断。

### Step 1: 扫描项目

#### 1.1 检测项目类型

读取 `package.json`，确认项目使用 Vue 3 + Vite + vue-router。如果不符合，告知用户当前 MVP 仅支持 Vue 3 + Vite 项目。

#### 1.2 提取路由配置

找到路由配置文件（通常在 `src/router/index.ts` 或 `src/router/index.js`），提取：

- 所有路由的 `name`、`path`、`component` 文件路径
- 路由模式：`createWebHashHistory()` → hash 模式，`createWebHistory()` → history 模式

#### 1.3 分析路由守卫

检查路由配置中的 `beforeEach` / `beforeEnter` 守卫：

- 如果守卫中读取了 `localStorage.getItem()`，记录 key 名称和期望的值类型
- 如果守卫中检查了特定条件（如 invite_code），记录 mock 方案

#### 1.4 分析 API 层

**前端 API 调用**：对每个路由对应的组件文件，检查 `axios` / `fetch` 调用的 URL 和方法，推断响应结构。

**后端检测**：检查项目是否有后端服务（如 `backend/`、`server/` 目录，或 `package.json` 中有后端启动脚本）。如果有后端：
- 找到后端启动方式（如 `go run main.go`、`npm run server`）
- 找到后端端口（从配置文件或 proxy 设置中读取）
- 找到登录接口：从 API 层代码中检测（如 `authApi.login`、`userApi.login`），提取 URL 和请求体字段
- 找到默认账号：从后端配置文件、种子数据（seed）、数据库或 README 中查找

**API Mock 策略**：
- **有后端项目**：截图前先启动后端，通过登录接口获取真实 token，注入 localStorage。不使用 API mock。
- **纯前端项目**（无后端）：分析 API 层的响应结构约定，生成 mock 数据用于截图。

#### 1.5 检测开发服务器端口

从 `vite.config.ts` 或 `vite.config.js` 中读取 `server.port`，默认为 5173。

### Step 2: 识别功能模块

#### 2.1 多信号源分析

综合以下信号识别项目的功能模块：

**信号 1: 路由分组（高置信度）**
- 路由前缀：`/admin/*` → admin 模块
- 嵌套路由结构：`children` 数组自然形成模块

**信号 2: 目录结构（高置信度）**
- `src/views/` 或 `src/pages/` 下的子目录
- 目录名通常就是模块名

**信号 3: 导航组件（中置信度）**
- 检查 `Sidebar`、`Nav`、`Menu` 等组件
- 菜单分组对应功能模块

**信号 4: README.md（辅助）**
- 功能介绍章节中的模块描述
- 用于给模块命名提供语义

#### 2.2 模块合并与命名

- 多个信号源指向同一组页面时，合并为一个模块
- 给每个模块一个有语义的中文名称（如"用户管理"、"内容发布"）
- 即使只有 1 个模块，也要展示路由列表让用户确认

### Step 3: 交互选择模块

如果 `video-config.yaml` 中已配置 `modules` 或 `pages` 字段，直接使用，跳过此步骤。

否则，**必须**向用户展示识别到的模块和路由，让用户确认要包含哪些页面。

**多模块场景**：展示模块列表，用户多选：

```
识别到以下功能模块：

  1. 用户流程 (3页) — input, generating, results
  2. 管理后台 (2页) — invite, admin

要为哪些模块生成视频？（可多选，输入编号如 "1,2" 或 "全部"）
```

**单模块场景**：展示路由列表，用户确认或排除：

```
识别到以下页面：

  1. input      — 输入页（取名表单）
  2. generating — 生成中（加载动画）
  3. results    — 结果页（姓名展示）
  4. invite     — 邀请码验证
  5. admin      — 管理后台

全部包含在视频中？或输入要排除的编号（如 "4,5" 表示排除 invite 和 admin）：
```

用户确认后：
1. 将选择结果写入 `video-config.yaml` 的 `modules` 字段
2. 后续步骤只处理选中模块的页面

### Step 4: 生成截图脚本

#### 4.1 检查前置依赖

检查用户的 `package.json` 是否包含 `puppeteer`：
- 如果没有，提示用户运行 `npm install puppeteer`
- 等待用户确认后再继续

#### 4.1.1 初始化 media 目录

在 `media/` 目录下生成 HyperFrames 所需的配置文件。

如果 `media/package.json` 已存在且包含 `check` 和 `render` 脚本，则跳过。否则生成：

```json
{
  "name": "media",
  "private": true,
  "type": "module",
  "scripts": {
    "check": "npx --yes hyperframes@latest lint && npx --yes hyperframes@latest validate && npx --yes hyperframes@latest inspect",
    "render": "npx --yes hyperframes@latest render"
  }
}
```

如果 `media/hyperframes.json` 已存在，则跳过。否则生成：

```json
{
  "$schema": "https://hyperframes.heygen.com/schema/hyperframes.json",
  "registry": "https://raw.githubusercontent.com/heygen-com/hyperframes/main/registry",
  "paths": {
    "blocks": "compositions",
    "components": "compositions/components",
    "assets": "assets"
  }
}
```

同时创建必要的空目录：`media/assets/`、`media/screenshots/`。

#### 4.2 生成脚本文件

在用户项目的 `media/` 目录下创建 `screenshot.mjs`。

**模板如下**（根据扫描结果填充占位变量）：

```javascript
import puppeteer from 'puppeteer'
import { mkdir } from 'fs/promises'
import { join, dirname } from 'path'
import { fileURLToPath } from 'url'

const __dirname = dirname(fileURLToPath(import.meta.url))
const SCREENSHOT_DIR = join(__dirname, 'screenshots')
const ASSETS_DIR = join(__dirname, 'assets')
const BASE_URL = '{{BASE_URL}}'

// 页面配置
const pages = [
  {{PAGES_ARRAY}}
]

// 支持增量截图：node screenshot.mjs --only=dashboard,manage
const onlyFilter = process.argv.find((arg) => arg.startsWith('--only'))
const filteredPages = onlyFilter
  ? pages.filter((p) => onlyFilter.split('=')[1]?.split(',').includes(p.name))
  : pages

{{AUTH_SETUP}}

async function main() {
  await mkdir(SCREENSHOT_DIR, { recursive: true })
  await mkdir(ASSETS_DIR, { recursive: true })

  {{AUTH_LOGIN}}

  const browser = await puppeteer.launch({
    headless: true,
    args: ['--no-sandbox', '--disable-setuid-sandbox'],
  })

  try {
    for (const { name, path } of filteredPages) {
      const page = await browser.newPage()
      await page.setViewport({ width: {{VIEWPORT_WIDTH}}, height: {{VIEWPORT_HEIGHT}} })

      {{AUTH_INJECT}}

      await page.goto(`${BASE_URL}${path}`, { waitUntil: 'networkidle2' })
      await page.waitForNetworkIdle({ idleTime: 500 }).catch(() => {})
      await new Promise((r) => setTimeout(r, 1500))

      const filePath = join(SCREENSHOT_DIR, `${name}.png`)
      const assetPath = join(ASSETS_DIR, `${name}.png`)
      await page.screenshot({ path: filePath, fullPage: true })
      await page.screenshot({ path: assetPath, fullPage: true })
      console.log(`Screenshot saved: ${filePath}`)

      await page.close()
    }
  } finally {
    await browser.close()
  }

  console.log(`\nAll screenshots saved to ${SCREENSHOT_DIR} and ${ASSETS_DIR}`)
}

main().catch((err) => {
  console.error('Screenshot failed:', err)
  process.exit(1)
})
```

#### 4.3 填充模板的规则

**{{BASE_URL}}**：从 `vite.config.ts` 的 `server.port` 读取，默认 `http://localhost:5173`

**{{VIEWPORT_WIDTH}} / {{VIEWPORT_HEIGHT}}**：从 `video-config.yaml` 读取，默认 1440 / 900

**{{AUTH_SETUP}} / {{AUTH_LOGIN}} / {{AUTH_INJECT}}**：根据项目类型选择以下方案之一填充。

---

**方案 A：有后端项目**（推荐）

截图前通过后端登录接口获取真实 token，注入 localStorage。需要后端和前端 dev server 都运行。

```javascript
// {{AUTH_SETUP}} — 留空（不需要）

// {{AUTH_LOGIN}} — 登录获取 token（LOGIN_URL 从 API 层代码中检测）
const authRes = await fetch('{{LOGIN_URL}}', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ {{LOGIN_BODY}} }),
})
const authJson = await authRes.json()
const authData = authJson.data  // 根据实际响应结构提取 token 和用户信息

// {{AUTH_INJECT}} — 注入 auth 数据
await page.evaluateOnNewDocument((auth) => {
  localStorage.setItem('token', auth.token)
  localStorage.setItem('userInfo', JSON.stringify(auth.user))
  localStorage.setItem('permissions', JSON.stringify(auth.user.permissions || []))
}, authData)
```

**前置条件**：
- 后端服务运行中（AI 应先检测后端端口是否可达，不可达则提示用户启动）
- 前端 dev server 运行中
- `{{LOGIN_URL}}`：从 API 层代码中检测登录接口（如 `/api/auth/login`、`/api/login`、`/api/user/login`）
- `{{LOGIN_BODY}}`：从登录页面组件中提取登录表单字段（如 `username, password` 或 `account, password`）
- 默认账号密码：AI 应从后端配置文件、种子数据或数据库中查找

---

**方案 B：纯前端项目**（无后端）

通过 Puppeteer 请求拦截 mock API 响应。注意：`setRequestInterception(true)` 可能与 Vite HMR 不兼容，如遇页面空白问题，应改用方案 A。

```javascript
// {{AUTH_SETUP}} — 定义 mock 数据和拦截函数
const API_MOCKS = { /* ... 见下方示例 ... */ }
function setupApiMocks(page) {
  page.on('request', (req) => {
    const url = req.url()
    if (!url.includes('/api/')) return req.continue()
    const path = new URL(url).pathname
    for (const [pattern, response] of Object.entries(API_MOCKS)) {
      if (new RegExp('^' + pattern.replace(/:[^/]+/g, '[^/]+$')).test(path)) {
        return req.respond({ status: 200, contentType: 'application/json', body: JSON.stringify(response) })
      }
    }
    return req.respond({ status: 200, contentType: 'application/json', body: JSON.stringify({ code: 200, data: null }) })
  })
}

// {{AUTH_LOGIN}} — 留空（不需要）

// {{AUTH_INJECT}} — 设置拦截 + 注入 auth
await page.evaluateOnNewDocument(() => {
  localStorage.setItem('token', 'mock-token')
  localStorage.setItem('userInfo', JSON.stringify({ id: 1, username: 'admin', is_admin: true }))
})
await page.setRequestInterception(true)
setupApiMocks(page)
```

**API_MOCKS 生成规则**：
1. 认证接口返回 mock token 和用户信息
2. 列表接口返回 3-5 条示例数据（字段名从组件代码推断）
3. 树形接口返回 2-3 层嵌套结构
4. 写入接口返回 `{ code: 200, data: null }`

---

**{{PAGES_ARRAY}}**：根据选中模块的路由生成。每项格式：
```javascript
{ name: '页面名称', path: '/路由路径' }  // history 模式
{ name: '页面名称', path: '#/路由路径' }  // hash 模式
```

### Step 5: 生成 HyperFrames Composition

#### 5.0 下载字体文件

HyperFrames 要求字体文件本地化（不支持 CDN 加载）。在 `media/assets/fonts/` 目录下准备中文字体：

**动态获取**（推荐）：先从 Google Fonts API 获取 CSS，解析出 woff2 URL，再下载：

```bash
mkdir -p media/assets/fonts

# 获取 Noto Sans SC 的 CSS（需要现代 User-Agent 才返回 woff2）
FONT_CSS=$(curl -s -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36" \
  "https://fonts.googleapis.com/css2?family=Noto+Sans+SC:wght@400;700&display=swap")

# 解析第一个 woff2 URL 用于 Regular (400)
REGULAR_URL=$(echo "$FONT_CSS" | grep -A2 "font-weight: 400" | grep -o 'https://[^)]*\.woff2' | head -1)

# 解析第一个 woff2 URL 用于 Bold (700)
BOLD_URL=$(echo "$FONT_CSS" | grep -A2 "font-weight: 700" | grep -o 'https://[^)]*\.woff2' | head -1)

# 下载字体文件
curl -o media/assets/fonts/NotoSansSC-Regular.woff2 "$REGULAR_URL"
curl -o media/assets/fonts/NotoSansSC-Bold.woff2 "$BOLD_URL"
```

**Fallback**：如果动态解析失败（网络问题、API 变更），使用预设的备用下载方式，提示用户手动从 Google Fonts 下载 Noto Sans SC 的 woff2 文件并放入 `media/assets/fonts/` 目录。

#### 5.1 检查前置依赖

检查用户的 `package.json` 是否包含 `hyperframes`：
- 如果没有，提示用户运行 `npm install hyperframes`
- 等待用户确认后再继续

#### 5.2 生成 composition 文件

在 `media/` 目录下创建 `index.html`。

**模板如下**（根据扫描结果和截图填充）：

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=1920, height=1080" />
    <script src="https://cdn.jsdelivr.net/npm/gsap@3.14.2/dist/gsap.min.js"></script>
    <style>
      /* 字体文件需要预先下载到 assets/fonts/ 目录 */
      @font-face {
        font-family: 'Noto Sans SC';
        font-weight: 400;
        font-style: normal;
        src: url('assets/fonts/NotoSansSC-Regular.woff2') format('woff2');
      }
      @font-face {
        font-family: 'Noto Sans SC';
        font-weight: 700;
        font-style: normal;
        src: url('assets/fonts/NotoSansSC-Bold.woff2') format('woff2');
      }
      * {
        margin: 0;
        padding: 0;
        box-sizing: border-box;
      }
      html, body {
        margin: 0;
        width: 1920px;
        height: 1080px;
        overflow: hidden;
        background: #0a0a0a;
      }
      body {
        font-family: "Noto Sans SC", "Inter", sans-serif;
      }

      .title-card {
        position: absolute;
        inset: 0;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
      }
      .title-card h1 {
        font-size: 72px;
        color: #e8d5b7;
        letter-spacing: 12px;
        text-shadow: 0 4px 20px rgba(232, 213, 183, 0.3);
      }
      .title-card .subtitle {
        font-size: 32px;
        color: #a0a0a0;
        margin-top: 24px;
        letter-spacing: 4px;
      }

      .page-slide {
        position: absolute;
        inset: 0;
        display: flex;
        align-items: center;
        justify-content: center;
        background: #0a0a0a;
      }
      .page-slide img {
        width: 1440px;
        height: auto;
        max-height: 960px;
        object-fit: contain;
        border-radius: 12px;
        box-shadow: 0 20px 60px rgba(0, 0, 0, 0.6);
      }

      .end-card {
        position: absolute;
        inset: 0;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
      }
      .end-card h2 {
        font-size: 56px;
        color: #e8d5b7;
        letter-spacing: 8px;
      }
      .end-card .tech-stack {
        font-size: 24px;
        color: #888;
        margin-top: 20px;
        letter-spacing: 2px;
      }
    </style>
  </head>
  <body>
    <div
      id="root"
      data-composition-id="main"
      data-start="0"
      data-duration="{{TOTAL_DURATION}}"
      data-width="1920"
      data-height="1080"
    >
      <!-- Title -->
      <div
        id="title-card"
        class="clip"
        data-start="0"
        data-duration="4"
        data-track-index="1"
      >
        <div class="title-card">
          <h1>{{TITLE}}</h1>
          <div class="subtitle">{{SUBTITLE}}</div>
        </div>
      </div>

      <!-- Page slides (动态生成) -->
      {{PAGE_SLIDES}}

      <!-- End card -->
      <div
        id="end-card"
        class="clip"
        data-start="{{END_START}}"
        data-duration="3"
        data-track-index="1"
      >
        <div class="end-card">
          <h2>{{END_TITLE}}</h2>
          <div class="tech-stack">{{TECH_STACK}}</div>
        </div>
      </div>
    </div>

    <script>
      window.__timelines = window.__timelines || {};
      const tl = gsap.timeline({ paused: true });

      // Title animation
      tl.from("#title-card h1", { opacity: 0, y: -40, duration: 1, ease: "power2.out" }, 0);
      tl.from("#title-card .subtitle", { opacity: 0, y: 20, duration: 0.8, ease: "power2.out" }, 0.4);
      tl.to("#title-card", { opacity: 0, duration: 0.5, ease: "power2.in" }, 3.5);
      tl.set("#title-card", { opacity: 0 }, 4.00);

      // Page slides fade in/out
      var slides = [
        {{SLIDES_TIMELINE}}
      ];

      slides.forEach(function (s) {
        tl.from(s.id, { opacity: 0, scale: 0.95, duration: 0.6, ease: "power2.out" }, s.start);
        tl.to(s.id, { opacity: 0, duration: 0.5, ease: "power2.in" }, s.start + 4.5);
        tl.set(s.id, { opacity: 0 }, s.start + 5);
      });

      // End card
      tl.from("#end-card", { opacity: 0, duration: 0.8, ease: "power2.out" }, {{END_START}});
      tl.from("#end-card h2", { opacity: 0, y: 30, duration: 0.6, ease: "power2.out" }, {{END_START}} + 0.2);
      tl.from("#end-card .tech-stack", { opacity: 0, duration: 0.5, ease: "power2.out" }, {{END_START}} + 0.6);

      window.__timelines["main"] = tl;
    </script>
  </body>
</html>
```

#### 5.3 填充模板的规则

**时长计算**：
- 标题页：固定 4 秒
- 每个页面幻灯片：固定 5 秒（含 0.5 秒过渡）
- 结束页：固定 3 秒
- 总时长 = 4 + (页面数 × 5) + 3

**{{TOTAL_DURATION}}**：总时长（秒）

**{{TITLE}}**：从 `video-config.yaml` 读取，或从 `package.json` name 推断，或使用项目目录名

**{{SUBTITLE}}**：从 `video-config.yaml` 读取，默认 "页面功能展示"

**{{PAGE_SLIDES}}**：为每个页面生成一个 clip，格式：
```html
<div
  id="slide-{{PAGE_NAME}}"
  class="clip"
  data-start="{{SLIDE_START}}"
  data-duration="5"
  data-track-index="1"
>
  <div class="page-slide">
    <img src="assets/{{PAGE_NAME}}.png" alt="{{PAGE_NAME}} page" />
  </div>
</div>
```

**{{SLIDE_START}}** 计算：4 + (页面索引 × 5)，第一个页面从 4 秒开始

**{{SLIDES_TIMELINE}}**：为每个页面生成 timeline 条目：
```javascript
{ id: "#slide-{{PAGE_NAME}}", start: {{SLIDE_START}} },
```

**{{END_START}}**：4 + (页面数 × 5)，结束页紧跟最后一个幻灯片

**{{END_TITLE}}**：同 {{TITLE}}

**{{TECH_STACK}}**：从 `package.json` 的 dependencies 推断，如 "Vue 3 + Vite + TypeScript"

**多模块过渡**：如果选中了多个模块，在模块之间可以插入一个短暂的模块标题过渡（可选，由 AI 根据页面数量决定是否需要）。

### Step 6: 执行截图

1. 自动检测 dev server 状态：`curl -s -o /dev/null -w "%{http_code}" {{BASE_URL}}`
   - 如果返回 200，dev server 已运行，直接进入下一步
   - 如果未响应，提示用户启动开发服务器：`npm run dev`，等待用户确认
2. **有后端项目**：自动检测后端服务状态（从 vite proxy 配置中读取后端地址）：
   - 如果后端可达，继续
   - 如果未响应，提示用户启动后端服务，等待用户确认
3. 执行截图脚本：`node media/screenshot.mjs`
4. 检查截图文件数量和大小（排除空白截图：文件小于 10KB 可能是空白页）
5. 如果截图异常（全部相同 MD5、文件过小），提示可能的原因：
   - 后端未运行 → 提示启动后端
   - API mock 不生效 → 建议改用方案 A（真实后端登录）
   - 页面路由不可访问 → 提示检查路由配置

### Step 7: 校验与渲染

1. 执行校验：在 `media/` 目录下运行 `npm run check`
2. 如果校验通过，执行渲染：`npm run render`
3. 渲染完成后，告知用户视频文件位置

### 错误处理

**Puppeteer 未安装**：
```
需要安装 Puppeteer 来生成截图。请运行：
npm install puppeteer
安装完成后告诉我，我继续执行。
```

**HyperFrames 未安装**：
```
需要安装 HyperFrames 来合成视频。请运行：
npm install hyperframes
安装完成后告诉我，我继续执行。
```

**开发服务器未启动**：
```
截图需要开发服务器运行中。请在另一个终端运行：
npm run dev
服务器启动后告诉我，我继续执行截图。
```

**截图失败**：
```
页面 "{{name}}" 截图失败：{{error}}
可能原因：
- 页面路由不可访问
- 页面有未处理的异步操作
- 需要特定的 localStorage 数据
请检查后重试。
```

### 最终输出

所有步骤完成后，告知用户：

```
视频生成完成！

生成的文件：
  media/screenshot.mjs    — 截图脚本（可重新运行）
  media/index.html         — HyperFrames composition
  media/screenshots/       — 页面截图
  media/assets/            — 视频引用的资源
  media/output/            — 输出的 MP4 视频

重新生成截图：node media/screenshot.mjs
重新截图指定页面：node media/screenshot.mjs --only=results,input
重新渲染视频：cd media && npm run render
```
