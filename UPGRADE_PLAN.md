# collaborative-docs-v2 升级计划

## 项目目标

以 `collaborative-text-editor`（已验证可运行的多人协作编辑器）为基座，吸取 `google-docs-crdt` 的架构优点（TypeScript、TipTap、磁盘持久化、版本历史），升级为生产级协作文档编辑器。

## 当前基座状态

- **前端**: React 19 + Vite 6 + Quill + Socket.IO + Yjs，纯 JavaScript
- **后端**: Express 5 + Socket.IO + Yjs WebSocket（单进程端口 3001）
- **持久化**: 内存（重启丢失）+ 客户端 IndexedDB
- **已验证**: 多人在线编辑正常工作，PDF/DOCX 导出、文档管理页、大文档分块传输均可用

## 升级方向

| 维度 | 当前 | 目标 |
|---|---|---|
| 语言 | JavaScript | TypeScript（strict 模式） |
| 编辑器内核 | Quill + y-quill | TipTap (ProseMirror) + @tiptap/extension-collaboration |
| 实时通道 | Socket.IO + Yjs WebSocket 双通道 | 纯 Yjs WebSocket（单通道，更简洁） |
| 持久化 | 内存 | 磁盘文件（.ydoc 二进制 + .meta.json） |
| 版本历史 | 无 | Yjs snapshot 快照 + 恢复 |
| React | 19 | 18（TipTap 生态兼容性更好） |

## 保留的 collaborative-text-editor 功能

- PDF 导出（html2pdf.js）
- DOCX 导出（file-saver）
- 文档管理页（DocsPage：列表/搜索/创建/删除）
- 大文档分块传输（需适配新架构）

## 不引入的 google-docs-crdt 功能（本次升级范围外）

- Chaos 测试面板（bug 较多，后续阶段再考虑）
- 分屏双端测试
- 活动流
- 光标碰撞检测

## 升级原则

1. **每阶段结束后项目必须可运行**——不破坏已有功能
2. **先架构后功能**——先把地基换好，再添砖加瓦
3. **不引入已知 bug**——google-docs-crdt 的问题要修复而非照搬
4. **保持简洁**——只做必要的改造，不过度设计

---

## 阶段一：核心架构升级（JS→TS + Quill→TipTap + Socket.IO→纯Yjs WS）

### 1.1 客户端基础设施迁移

#### 1.1.1 初始化 TypeScript + React 18 + TipTap 依赖

**文件**: `client/package.json`

- 降级 React 18→19 为 18.3（TipTap 2.11 生态兼容性最佳）
- 新增 TypeScript 工具链: `typescript`, `@types/react`, `@types/react-dom`, `@types/node`
- 移除 Quill 相关: `quill`, `quill-cursors`, `y-quill`
- 新增 TipTap 相关: `@tiptap/react`, `@tiptap/starter-kit`, `@tiptap/pm`, `@tiptap/extension-collaboration`, `@tiptap/extension-collaboration-cursor`, `@tiptap/extension-underline`, `@tiptap/extension-text-align`, `@tiptap/extension-text-style`, `@tiptap/extension-color`, `@tiptap/extension-highlight`, `@tiptap/extension-placeholder`
- 移除 Socket.IO: `socket.io-client`
- 保留: `yjs`, `y-websocket`, `y-indexeddb`, `lucide-react`, `docx`, `file-saver`, `html2pdf.js`
- Vite 构建脚本改为 `tsc && vite build`

#### 1.1.2 TypeScript 配置

**新建文件**: `client/tsconfig.json`, `client/tsconfig.node.json`

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true
  },
  "include": ["src"]
}
```

#### 1.1.3 Vite 配置更新

**文件**: `client/vite.config.js` → `client/vite.config.ts`

- 添加 Vite WebSocket 代理（/yjs 路径代理到后端 3001）
- 移除 Socket.IO 代理（不再需要）

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      '/yjs': {
        target: 'ws://localhost:3001',
        ws: true,
        changeOrigin: true,
      },
    },
  },
})
```

### 1.2 客户端源码迁移

#### 1.2.1 类型定义

**新建文件**: `client/src/types/index.ts`

```typescript
// 用户 awareness 状态
export interface UserAwareness {
  id: string
  name: string
  color: string
}

// 协作会话
export interface CollabSession {
  doc: Y.Doc
  provider: WebsocketProvider
  persistence: IndexeddbPersistence
  user: UserAwareness
  roomName: string
  destroy: () => void
}

// 文档元数据
export interface DocumentMeta {
  id: string
  name: string
  createdAt: number
  updatedAt: number
}

// 连接状态
export type ConnectionStatus = 'connecting' | 'connected' | 'disconnected'
```

#### 1.2.2 Yjs Provider 服务

**文件**: `client/src/services/yjsProvider.js` → `client/src/services/yjsProvider.ts`

- 移除 Socket.IO 依赖，只保留 Yjs WebSocket Provider
- 添加 TypeScript 类型
- WS_URL 从环境变量读取，dev 模式通过 Vite 代理连接后端 3001

```typescript
import * as Y from 'yjs'
import { WebsocketProvider } from 'y-websocket'
import { IndexeddbPersistence } from 'y-indexeddb'
import type { CollabSession, UserAwareness } from '../types'

const WS_URL = import.meta.env.VITE_YJS_URL || 'ws://localhost:3001/yjs'

export function createYjs(
  docId: string,
  customUser?: Partial<UserAwareness>
): CollabSession {
  const ydoc = new Y.Doc()
  const persistence = new IndexeddbPersistence(docId, ydoc)
  const provider = new WebsocketProvider(WS_URL, docId, ydoc)

  const user: UserAwareness = customUser
    ? { id: customUser.id || ..., name: customUser.name || ..., color: customUser.color || ... }
    : { /* 随机生成 */ }

  provider.awareness.setLocalStateField('user', user)

  const destroy = () => {
    provider.destroy()
    persistence.destroy()
    ydoc.destroy()
  }

  return { doc: ydoc, provider, persistence, user, roomName: docId, destroy }
}
```

#### 1.2.3 Storage 服务

**文件**: `client/src/services/storage.js` → `client/src/services/storage.ts`

- 添加 TypeScript 类型（DocumentMeta 接口）
- 逻辑不变

#### 1.2.4 Editor 组件重写

**文件**: `client/src/components/Editor.jsx` → `client/src/components/Editor.tsx`

这是最大的改动。核心替换：

| 旧（Quill） | 新（TipTap） |
|---|---|
| `new Quill(container, {...})` | `useEditor({ extensions: [...] })` |
| `QuillBinding(ytext, quill, awareness)` | `Collaboration.configure({ document: session.doc })` + `CollaborationCursor.configure({ provider, user })` |
| `quill.on('text-change', ...)` | `editor.on('update', ...)` 或 `onUpdate` 回调 |
| `quill.getModule('cursors')` | `CollaborationCursor` 扩展自动处理 |
| 手动 toolbar container | `<GoogleDocsToolbar editor={editor} />` 组件 |
| `quill.getText()` | `editor.getText()` |

**需要新建的子组件**:

- `client/src/components/EditorToolbar.tsx`——从 Editor.jsx 中拆出工具栏
  - 字体选择（13 种 Google Fonts）
  - 字号选择（8pt-72pt，15 档）
  - 粗体/斜体/下划线/删除线
  - 文字颜色/背景色
  - 对齐方式
  - 有序/无序列表
  - 缩进
  - 链接
  - 清除格式

  注意：TipTap 的字体/字号需要自定义扩展（TipTap 没有内置 font-family/font-size 扩展），可以：
  - 方案 A: 使用 `@tiptap/extension-font-family`（社区扩展）+ 自定义 FontSize 扩展
  - 方案 B: 用 TextStyle + 自定义属性
  - 建议方案 A，更标准化

**Editor.tsx 保留的功能**:
- 文档标题编辑 + awareness 同步
- 协作者头像列表
- 连接状态指示器
- 编辑/查看模式切换
- PDF / DOCX 导出（适配 TipTap 的 DOM 结构）
- 字数统计

**Editor.tsx 移除的功能**:
- Socket.IO 连接（`io(API_URL)`, `join-document`, `load-document-chunk` 等）
- Quill 相关的所有代码
- 手动 cursor 管理（由 CollaborationCursor 扩展接管）

#### 1.2.5 DocsPage 组件迁移

**文件**: `client/src/components/DocsPage.jsx` → `client/src/components/DocsPage.tsx`

- 添加 TypeScript 类型
- 逻辑基本不变（localStorage 文档管理）
- 移除对 Socket.IO 的任何引用（如果有）

#### 1.2.6 App 和 main 迁移

**文件**: `client/src/App.jsx` → `client/src/App.tsx`, `client/src/main.jsx` → `client/src/main.tsx`

- 添加 TypeScript 类型
- 路由结构不变

#### 1.2.7 generateId 工具迁移

**文件**: `client/src/utils/generateId.js` → `client/src/utils/generateId.ts`

- 添加 TypeScript 类型

#### 1.2.8 CSS 适配

**文件**: `client/src/index.css`

- 移除 Quill 专属样式（`.ql-editor`, `.ql-toolbar`, `.ql-snow` 等）
- 添加 TipTap / ProseMirror 样式（`.ProseMirror`, `.tiptap` 等）
- 保留设计系统的 CSS 变量和通用样式
- 保留 Google Fonts 导入（TipTap 编辑器也需要这些字体）

### 1.3 服务端迁移

#### 1.3.1 TypeScript 化

**新建文件**: `server/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "CommonJS",
    "moduleResolution": "node",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"]
}
```

#### 1.3.2 server/index.js → server/src/server.ts

**文件**: `server/index.js` → `server/src/server.ts`

核心改动：
- **移除 Socket.IO**——不再需要 `socket.io` 和 `socket.io-client`
  - `join-document`, `send-changes`, `save-document`, `load-document-chunk` 事件全部移除
  - 文档同步完全由 Yjs WebSocket 的 `setupWSConnection` 处理
- **移除内存文档存储**——`documents` 对象和 `documentOrder` 数组删除
  - 持久化由阶段二的 DiskPersistence 接管
  - 阶段一暂时只用 Yjs WebSocket 的内存同步（无持久化，重启丢失——与原行为一致）
- **保留** Express + health 端点
- **保留** Yjs WebSocket 服务（`/yjs` 路径）
- **保留** CORS 配置
- **保留** 优雅关闭

```typescript
import express from 'express'
import http from 'http'
import cors from 'cors'
import { WebSocketServer, WebSocket } from 'ws'
import { setupWSConnection } from 'y-websocket/bin/utils'

const PORT = process.env.PORT || 3001
const CLIENT_ORIGIN = process.env.CLIENT_ORIGIN || 'http://localhost:5173'

const app = express()
app.use(cors({ origin: CLIENT_ORIGIN, methods: ['GET', 'POST'] }))

app.get('/health', (_req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() })
})

const server = http.createServer(app)
const wss = new WebSocketServer({ noServer: true, perMessageDeflate: true })

wss.on('connection', setupWSConnection)

server.on('upgrade', (request, socket, head) => {
  const pathname = new URL(request.url, 'http://x').pathname
  if (pathname === '/yjs') {
    wss.handleUpgrade(request, socket, head, (ws) => {
      wss.emit('connection', ws, request)
    })
  }
})

server.listen(PORT, () => {
  console.log(`[server] Running on http://localhost:${PORT}`)
})
```

#### 1.3.3 server/package.json 更新

- 移除 `socket.io`
- 添加 TypeScript 工具链: `typescript`, `@types/node`, `@types/express`, `@types/cors`, `@types/ws`, `ts-node`
- 修改 scripts: `dev: ts-node src/server.ts`, `build: tsc`, `start: node dist/server.js`

### 1.4 阶段一验收标准

- [ ] `npm run build` 在 client 和 server 都成功（TypeScript 编译无错误）
- [ ] `npm run dev` 启动后端（3001）和前端（5173）
- [ ] 浏览器打开 http://localhost:5173，文档列表页正常显示
- [ ] 创建文档、删除文档、搜索文档正常工作
- [ ] 进入编辑器，TipTap 工具栏完整显示（字体/字号/粗斜体/颜色/对齐/列表等）
- [ ] 富文本编辑正常（所有格式按钮可用）
- [ ] 开两个浏览器标签打开同一文档，实时同步正常
- [ ] 远程光标显示正常（颜色 + 用户名）
- [ ] 协作者头像列表正常显示
- [ ] 连接状态指示器正常（Connected/Offline）
- [ ] PDF 导出正常
- [ ] DOCX 导出正常
- [ ] 字数统计正常
- [ ] 文档标题编辑 + 同步正常
- [ ] 编辑/查看模式切换正常
- [ ] IndexedDB 离线缓存正常（刷新页面内容不丢失）

---

## 阶段二：磁盘持久化 + 版本历史

### 2.1 磁盘持久化（从 google-docs-crdt 移植 + 修复）

#### 2.1.1 服务端持久化层

**新建文件**: `server/src/persistence.ts`

从 `google-docs-crdt/server/src/persistence.ts` 移植，修复以下问题：
- `fs.writeFileSync` → `fs.promises.writeFile`（异步写入，不阻塞事件循环）
- `ydoc.on('update')` 监听器泄漏 → 在 `writeState` 回调中清理
- 添加 debounce timer 清理逻辑

```typescript
import * as fs from 'fs'
import * as path from 'path'
import * as Y from 'yjs'

const STORAGE_DIR = path.resolve(__dirname, '../storage')

// 确保存储目录存在
if (!fs.existsSync(STORAGE_DIR)) {
  fs.mkdirSync(STORAGE_DIR, { recursive: true })
}

interface DocumentMetadata {
  id: string
  title: string
  createdAt: string
  updatedAt: string
  updateCount: number
}

const writeDebounceTimers = new Map<string, NodeJS.Timeout>()

export class DiskPersistence {
  // ... 移植自 google-docs-crdt，修复异步写入和监听器泄漏
}
```

#### 2.1.2 服务端集成持久化

**文件**: `server/src/server.ts`

- 添加 `setPersistence` 到 y-websocket 的 `setupWSConnection`
- 添加 REST API 端点:
  - `GET /api/documents`——列出所有文档
  - `GET /api/documents/:id/metadata`——获取文档元数据
  - `PATCH /api/documents/:id/metadata`——更新文档标题

#### 2.1.3 客户端适配

**文件**: `client/src/components/DocsPage.tsx`

- 文档列表从 localStorage 改为调用 `GET /api/documents`（同时保留 localStorage 作为离线缓存）
- 文档创建/删除时同步到服务端

### 2.2 版本历史（从 google-docs-crdt 移植 + 修复）

#### 2.2.1 修复方案

google-docs-crdt 的版本历史只存元数据，不能恢复。修复方案：
- 使用 Yjs 的 `Y.encodeStateAsUpdate(doc)` 保存完整 CRDT 状态
- 存储位置：IndexedDB（客户端本地，因为版本历史是个人操作记录）
- 添加恢复功能：`Y.applyUpdate(doc, snapshot)` 将文档回滚到指定版本

#### 2.2.2 客户端版本历史服务

**新建文件**: `client/src/services/versionHistory.ts`

```typescript
import * as Y from 'yjs'
import { getDoc, setDoc } from 'idb-keyval' // 或直接用 IndexedDB API

export interface VersionSnapshot {
  id: string
  name: string
  timestamp: number
  author: string
  crdtBytes: Uint8Array  // 完整 CRDT 状态（关键修复）
  preview: string
}

// 保存快照
export async function createSnapshot(doc: Y.Doc, name: string, author: string): Promise<VersionSnapshot>

// 加载快照列表
export async function listSnapshots(roomName: string): Promise<VersionSnapshot[]>

// 恢复到指定版本
export async function restoreSnapshot(snapshot: VersionSnapshot, doc: Y.Doc): Promise<void>
```

#### 2.2.3 版本历史 UI

**新建文件**: `client/src/components/VersionHistoryModal.tsx`

- 从 google-docs-crdt 移植 UI 结构
- 添加"恢复到此版本"按钮
- 样式适配现有设计系统（不用 google-docs-crdt 的白色主题，适配 collaborative-text-editor 的风格）

### 2.3 阶段二验收标准

- [ ] 服务端重启后文档内容不丢失
- [ ] `GET /api/documents` 返回正确的文档列表
- [ ] 文档标题更新同步到服务端
- [ ] 版本历史模态框正常显示
- [ ] 可以创建命名快照
- [ ] 可以恢复到之前的版本（文档内容实际回滚）
- [ ] 快照存储在 IndexedDB 中，刷新页面不丢失

---

## 阶段三：大文档分块传输 + 收尾优化（后续）

### 3.1 大文档分块传输

- 评估 TipTap + Yjs 在大文档场景下的性能
- 如有需要，在 Yjs WebSocket 层实现分块传输（替代原 Socket.IO 方案）

### 3.2 代码质量

- ESLint + Prettier 配置
- 移除所有 `any` 类型
- 添加关键模块的单元测试

### 3.3 文档

- 更新 README.md
- 添加架构图
- 添加开发指南

---

## 技术风险与注意事项

### 风险 1: TipTap 字体/字号扩展

TipTap 没有内置的 font-family 和 font-size 扩展。需要：
- 使用 `@tiptap/extension-font-family`（官方社区扩展）处理字体
- 自定义 FontSize 扩展处理字号（继承 TextStyle，添加 `fontSize` 属性）

如果社区扩展不满足需求，需要自己写 ProseMirror 扩展——这是阶段一最大的技术风险。

### 风险 2: React 18 vs 19 降级

从 React 19 降级到 18 可能导致一些 API 不兼容（如 `useFormStatus`, `use()` 等）。但 collaborative-text-editor 当前代码用的是基础 hooks（useState, useEffect, useRef, useCallback），降级风险很低。

### 风险 3: y-websocket 版本

collaborative-text-editor 用 `y-websocket@^1.5.0`（CommonJS），google-docs-crdt 用 `y-websocket@^2.0.4`。需要统一版本。建议用 v2（更新版本，与 TipTap 生态更兼容），但需要验证 `setupWSConnection` API 是否有变化。

### 注意事项: StrictMode

collaborative-text-editor 的 README 明确提到不用 StrictMode（会导致 Yjs provider 双重创建）。升级后继续保持不用 StrictMode，或者用 ref guard 防止双重初始化。

---

## 文件变更清单

### 阶段一变更

| 操作 | 文件路径 |
|---|---|
| 修改 | `client/package.json` |
| 修改 | `client/vite.config.js` → `client/vite.config.ts` |
| 新建 | `client/tsconfig.json` |
| 新建 | `client/tsconfig.node.json` |
| 新建 | `client/src/types/index.ts` |
| 重写 | `client/src/services/yjsProvider.ts` |
| 修改 | `client/src/services/storage.ts` |
| 重写 | `client/src/components/Editor.tsx` |
| 新建 | `client/src/components/EditorToolbar.tsx` |
| 修改 | `client/src/components/DocsPage.tsx` |
| 修改 | `client/src/App.tsx` |
| 修改 | `client/src/main.tsx` |
| 修改 | `client/src/utils/generateId.ts` |
| 修改 | `client/src/index.css` |
| 修改 | `server/package.json` |
| 新建 | `server/tsconfig.json` |
| 重写 | `server/src/server.ts` |
| 删除 | `client/src/components/Editor.jsx` |
| 删除 | `client/src/components/DocsPage.jsx` |
| 删除 | `client/src/App.jsx` |
| 删除 | `client/src/main.jsx` |
| 删除 | `client/src/services/yjsProvider.js` |
| 删除 | `client/src/services/storage.js` |
| 删除 | `client/src/utils/generateId.js` |
| 删除 | `server/index.js` |

### 阶段二新增

| 操作 | 文件路径 |
|---|---|
| 新建 | `server/src/persistence.ts` |
| 新建 | `client/src/services/versionHistory.ts` |
| 新建 | `client/src/components/VersionHistoryModal.tsx` |
| 修改 | `server/src/server.ts`（集成持久化） |
| 修改 | `client/src/components/Editor.tsx`（添加版本历史入口） |
| 修改 | `client/src/components/DocsPage.tsx`（文档列表从服务端获取） |
