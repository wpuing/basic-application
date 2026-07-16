# AGENTS.md

> 前端（Web）编码约束。目录按业务域垂直切分；规范对齐 Vue 官方风格指南 + TypeScript 严格模式实践。  
> 占位符 `[app]` / `{biz}` 按实际项目替换。  
> 与后端配合时，接口契约以后端 `AGENTS.md` / OpenAPI 为准；本文件只约束前端。

## 1. 项目概览

- **技术栈**：Vue 3 + TypeScript + Vite + Vue Router + Pinia + Axios（UI 库按需：Element Plus / Ant Design Vue）
- **架构**：按业务域（feature）垂直切分 + 共享层；前后端分离
- **工程形态（推荐）**：

  | 目录/包 | 职责 |
  |--------|------|
  | `src/views|{biz}` / `src/features/{biz}` | 页面与业务域 UI |
  | `src/api` | HTTP 接口封装 |
  | `src/stores` | 全局/跨页状态（Pinia） |
  | `src/components` | 通用展示组件（无业务） |
  | `src/composables` | 可复用组合式逻辑 |
  | `src/utils` / `src/constants` | 工具与常量 |

- **后端**：只通过 REST/OpenAPI 交互；禁止在前端写死服务端权限逻辑作为唯一校验

## 2. 目录结构

根目录应用名：`[app]`（如 `order-web`）。先按层/职责，业务代码再按 `{biz}` 切分。

```text
[app]/
├── index.html
├── package.json
├── vite.config.ts
├── tsconfig.json
├── .env.development / .env.production
├── public/
└── src/
    ├── main.ts                      # 应用入口
    ├── App.vue
    ├── env.d.ts
    │
    ├── api/                         # 基础设施：HTTP
    │   ├── request.ts               # Axios 实例、拦截器、统一错误
    │   ├── types.ts                 # 通用 Result / PageResult
    │   └── {biz}/
    │       └── index.ts             # 该域 API 方法（仅调后端）
    │
    ├── stores/                      # 应用状态（Pinia）
    │   ├── user.ts                  # 登录用户、token、权限码
    │   └── {biz}.ts                 # 跨页共享的业务状态（能局部则不放全局）
    │
    ├── router/
    │   ├── index.ts                 # 路由表
    │   └── guards.ts                # 登录/权限守卫
    │
    ├── views/                       # 页面（路由落地）
    │   └── {biz}/
    │       ├── list.vue             # 列表页
    │       ├── detail.vue
    │       └── components/          # 仅本域复用的页面子组件
    │
    ├── components/                  # 全局通用组件（无具体业务）
    │   ├── PageTable/
    │   └── StatusTag/
    │
    ├── composables/                 # 组合式函数 useXxx
    │   ├── useTable.ts
    │   └── usePermission.ts
    │
    ├── constants/
    │   ├── enums.ts
    │   └── storage-keys.ts
    │
    ├── utils/
    │   ├── format.ts
    │   └── validate.ts
    │
    ├── styles/                      # 全局样式、变量
    └── assets/
```

可选：复杂域用 `src/features/{biz}/` 聚合该域的 `api` / `components` / `composables` / `views`，与后端「按 biz 垂直切分」对齐。

### 调用链

```text
View / 页面组件
  → composable / store（编排、缓存）
  → api/{biz}（参数组装）
  → request.ts（Axios）
  → 后端 *Ctl
```

### 层职责与依赖

```text
views → composables / stores → api → request
          ↓              ↓
      components ←── constants / utils
```

| 层 | 允许 | 禁止 |
|----|------|------|
| **views** | 页面布局、拼装组件、调 composable/store/api | 直接写 Axios；塞复杂纯函数（下沉 utils） |
| **components** | 可复用 UI，通过 props/emits 通信 | 依赖具体路由、具体业务 store |
| **composables** | 可复用有状态逻辑、副作用封装 | 耦合某个页面私有 DOM 结构 |
| **stores** | 跨路由共享状态、持久化用户信息 | 把所有接口结果都塞进全局 store |
| **api** | 定义 URL、方法、入参出参类型 | UI 提示、路由跳转、操作 DOM |

### 命名约定

| 类型 | 约定 | 示例 |
|------|------|------|
| 页面 | 小写短横或业务语义文件名 | `list.vue`、`OrderList.vue`（二选一，项目内统一） |
| 组件 | PascalCase 文件/目录 | `StatusTag/index.vue` |
| 组合式函数 | `use` + PascalCase | `useOrderTable.ts` |
| API 方法 | 动词 + 名词 | `fetchOrderPage`、`createOrder` |
| Store | 名词 | `useOrderStore` |
| 类型/接口 | PascalCase；请求 `XxxQuery`，响应 `XxxVO` / `XxxDTO` | `OrderQuery`、`OrderVO` |
| 常量 | UPPER_SNAKE | `TOKEN_KEY` |
| 枚举 | PascalCase + 成员语义清晰 | `OrderStatus.Pending` |

### 新功能清单

1. `api/types` 或 `api/{biz}` 对齐后端 VO / QueryVO  
2. `api/{biz}/index.ts` 封装接口  
3. 需要跨页状态时加 `stores/{biz}.ts`  
4. `views/{biz}/` 页面 + 域内 `components/`  
5. 路由注册 + `guards` 权限（如需）  
6. 可复用逻辑抽 `composables/useXxx.ts`  

## 3. HTTP / 接口约定

- 统一 `request.ts`：baseURL 来自环境变量；超时默认 30s（上传/导出可单独加长）
- 请求头：`Authorization: Bearer <token>`；可选 `X-Request-Id`
- 响应约定与后端一致，例如：

```ts
interface ApiResult<T> {
  code: number
  message: string
  data: T
}

interface PageResult<T> {
  list: T[]
  total: number
  page: number
  pageSize: number
}
```

- 业务失败：拦截器统一处理 `code !== 成功码`；页面可再 catch 做局部提示
- 401：清 token、跳登录；禁止死循环重试
- 禁止在组件内硬编码完整 URL；禁止 `any` 抹平响应类型（必要时用 `unknown` + 收窄）
- 列表查询防抖；同一查询进行中取消上一次（AbortController）按需启用
- 上传/下载走独立方法，注意 `blob` 与文件名解析

## 4. 状态与数据

- 默认：**服务端状态**用请求 + 组件/ composable 本地 ref；仅跨页、用户会话进 Pinia
- Token、用户信息存 Pinia + `localStorage`/`sessionStorage`（键名进 `constants`）
- 列表筛选条件：可同步到 query（利于刷新/分享），注意类型与默认值
- 表单：编辑态与提交 DTO 分离；提交前校验，失败定位字段
- 金额、日期展示用 `utils/format`；禁止页面魔法格式串散落
- 枚举与后端字典对齐，集中在 `constants/enums.ts` 或接口下发字典

## 5. 编码规范

> Vue 3 官方风格指南 + TypeScript `strict`；建议 ESLint（`eslint-plugin-vue`）+ Prettier。  
> IDE 开启 Volar（Vue - Official），禁用 Vetur。

### 5.1 命名与文件

- 禁止拼音英文混用、禁止无意义缩写
- 组件名多词（避免与 HTML 冲突）：`UserCard` 而非 `User`
- 布尔 props / 变量：`is`/`has`/`can`/`should` 前缀
- 事件名 kebab-case：`update:modelValue`、`row-click`
- 私有约定：模块内 `_` 前缀仅用于真正内部、不导出的符号（慎用）

### 5.2 TypeScript

- 开启 `strict`；禁止随意 `any`、禁止 `as any` 逃逸（确需断言写注释原因）
- Props / Emits 用类型声明；复杂对象抽 `types.ts`
- 接口与类型别名：对象形状用 `interface`，联合/工具类型用 `type`
- 枚举优先 `as const` 对象或字符串联合，避免数字枚举难读

### 5.3 Vue 组件

- 优先 `<script setup lang="ts">` + Composition API
- props 向下、events 向上；跨多层用 provide/inject 或 store，避免万能 event bus
- `v-for` 必须有稳定 `:key`（禁用 index 作为会重排列表的唯一 key）
- 不要在模板里写复杂表达式，算进 `computed`
- 副作用放 `watch` / `onMounted`；组件卸载取消请求、移除监听
- 大列表考虑虚拟滚动；图片懒加载
- 样式默认 `scoped`；全局类放 `styles/`，慎用 `!important`

### 5.4 常量与魔法值

- 禁止魔法数字/字符串；状态码、存储 key、权限码进常量
- 环境区分只用 `import.meta.env`，禁止在业务里写死域名

### 5.5 控制与复杂度

- 单文件建议 ≤300 行；过大拆子组件或 composable
- 嵌套判断用早期 return；复杂权限/状态机抽函数或表驱动

### 5.6 注释

- 公共 composable、复杂业务组件写清用途与边界
- 注释与代码同步；删除无效注释
- `TODO` 带负责人或工单号（团队约定）

### 5.7 异常与用户提示

- 网络错误与业务错误文案区分；避免直接把后端堆栈展示给用户
- 按钮提交加 loading / 防重复点击
- 路由懒加载失败给降级提示

### 5.8 日志与调试

- 生产构建去掉 `console.debug`；必要埋点走统一 logger
- 禁止日志输出 token、密码、证件号等敏感信息

### 5.9 安全【强制】

- 不信任前端权限：按钮可隐藏，接口仍须后端鉴权
- 防 XSS：慎用 `v-html`；必须用时消毒
- 开放重定向：登录回跳校验白名单域名/相对路径
- 密钥、AppSecret 禁止进前端仓库；仅用发布时可公开的配置
- 第三方脚本与 CDN 评估完整性（SRI）按团队要求

### 5.10 测试【推荐】

- 纯函数 / composable 用 Vitest
- 关键交互用 Vue Test Utils 或 Playwright/Cypress（按项目）
- 工具函数边界值、空值必测

## 6. 路由与权限

- 路由 `meta`：`title`、`requiresAuth`、`roles` / `permissions`
- 守卫顺序：登录校验 → 权限校验 → 动态标题
- 动态路由：按权限过滤菜单与路由表，放 store 统一来源
- 404 / 403 独立页；无权限不暴露隐藏路由的组件资源（仍须后端兜底）

## 7. UI 与体验

- 间距、字号、色彩走设计变量（CSS 变量或 UI 主题），禁止页面随意色值堆砌
- 表单：必填、长度、格式与后端校验对齐；错误信息可读
- 空态、加载态、错误态三态齐全
- 无障碍：按钮有语义、图标按钮有 `aria-label`
- 国际化：文案进 i18n 或统一文案表（若项目需要）

## 8. 工程与质量

| 项 | 要求 |
|----|------|
| Node | 与 `.nvmrc` / `engines` 一致（建议 LTS） |
| 包管理 | 锁定 pnpm / npm / yarn 之一；提交 lockfile |
| 脚本 | `dev` / `build` / `lint` / `typecheck` / `test` |
| 提交前 | lint + `vue-tsc --noEmit`（或 CI 等价） |
| 环境变量 | 仅 `VITE_` 前缀暴露给客户端 |

## 9. 性能关注点

| 优先级 | 风险 | 要求 |
|--------|------|------|
| P0 | 首屏 | 路由懒加载；大依赖按需引入（UI/图表） |
| P0 | 请求 | 禁瀑布流无意义串行；列表分页；防重复提交 |
| P1 | 渲染 | 大表虚拟滚动；稳定 key；避免多余 deep watch |
| P1 | 包体积 | 分析 bundle；去掉无用 polyfill |
| P2 | 体验 | 骨架屏/按需 loading；图片压缩与懒加载 |
| P2 | 合规 | 埋点与日志脱敏 |

## 10. 与后端协作

- 字段命名：前端 TS 可用 camelCase，若后端 snake_case，在 api 层转换，勿污染全项目
- 变更接口同步更新类型与 Mock；禁止页面里留「临时 any」长期不改
- 分页、排序、筛选参数与后端 QueryVO 对齐
- 时间统一 ISO 或约定时间戳，展示层再格式化
