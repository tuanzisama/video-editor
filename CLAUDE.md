# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

video-clip — 纯浏览器端视频剪辑工具。基于 Vue 3 + Vite + TypeScript + TDesign Vue Next，核心剪辑能力由 FFmpeg WebAssembly (@ffmpeg/ffmpeg 0.12.x + @ffmpeg/core-mt) 在浏览器中完成，无服务端依赖。

## 常用命令

```bash
pnpm dev          # 启动开发服务器
pnpm build        # 类型检查 + 构建 (vue-tsc -b && vite build)
pnpm preview      # 预览构建产物
```

## 架构

### 技术栈

| 层 | 选型 |
|---|---|
| 框架 | Vue 3 (Composition API + `<script setup>`) |
| 构建 | Vite 5 + vue-tsc |
| UI 组件库 | TDesign Vue Next |
| CSS 预处理 | Sass/SCSS (scoped) |
| 视频处理 | @ffmpeg/ffmpeg 0.12.10 + @ffmpeg/core-mt (多线程 WASM) |
| 工具库 | dayjs, lodash-es, pretty-bytes, file-saver |
| 包管理 | pnpm |

### 文件结构

```
src/
├── main.ts              # 入口: 挂载 Vue 应用 + TDesign + reset.css
├── App.vue              # 唯一业务组件: 视频剪辑全流程
├── components/          # 子组件 (当前只有 HelloWorld 模板残留)
├── index.css            # 全局样式 (字体)
└── lib.es2017.typedarrays.d.ts  # TypedArray 联合类型声明 (ffmpeg readFile 返回值)
```

核心逻辑全部在 `App.vue` (<script setup>) 中:

1. **文件选择** → 创建 `<input type="file" accept="video/*">`，选择后生成 Blob URL 供 `<video>` 预览
2. **时间范围剪辑** → `<t-slider range>` 选择起止时间，拖动时联动预览区 `currentTime`
3. **FFmpeg 转码** → `FFmpeg.load()` 从 unpkg CDN 加载 core-mt WASM 资源，`ffmpeg.exec()` 执行 `-ss/-to/-c copy` 无损切割，`ffmpeg.readFile()` 输出 Blob
4. **下载** → `file-saver` 触发浏览器下载

### COOP/COEP 跨域隔离

FFmpeg WASM 多线程依赖 `SharedArrayBuffer`，需要两个 HTTP 响应头:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

- **开发环境**: `vite.config.ts` 的 `server.headers` 配置
- **生产环境**: `vercel.json` 的 `headers` 配置

### FFmpeg 加载策略

FFmpeg 实例在首次点击"开始"时懒加载，通过 `shallowRef` 持有单例。WASM 资源通过 `toBlobURL()` 绕过 CORS，从 `https://unpkg.com/@ffmpeg/core-mt@0.12.6/dist/esm` 加载。`vite.config.ts` 的 `optimizeDeps.exclude` 中排除了 `@ffmpeg/ffmpeg` 和 `@ffmpeg/util`，避免 Vite 预构建时出现问题。
