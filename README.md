# basic-application

面向 **Java 后端** 与 **Web 前端** 的 AI 编码约束模板：通过各端的 `AGENTS.md` 统一 Agent / 开发者在分层、接口、编码与安全方面的约定。

## 仓库结构

```text
basic-application/
├── README.md                 # 本说明
├── agent-app/
│   └── AGENTS.md             # Java 后端 AI 编码入口
└── web-app/
    └── AGENTS.md             # Vue3 前端 AI 编码入口
```

按目标工程类型，将对应目录下的 `AGENTS.md` 复制到仓库根目录（或与 Cursor / 其他 Agent 约定的路径），并替换占位符即可。

## 模板说明

### agent-app（后端）

- **DDD 包结构**：`interfaces` → `application` → `domain` ← `infrastructure`
- **命名约定**：`Ctl` / `Asvc` / `Dsvc` / `Repository` / `Mapper`
- **数据库与性能**：通用字段、索引、分页、外部调用超时与熔断
- **编码规范**：《阿里巴巴 Java 开发手册（黄山版）》要点
- **认证与 API 文档**：JWT / Security、Knife4j 等摘要

### web-app（前端）

- **目录结构**：按业务域垂直切分 + `api` / `stores` / `views` / `composables`
- **调用链**：View → composable/store → api → Axios → 后端
- **HTTP / 路由权限 / 安全**：统一拦截器、守卫、XSS 与权限边界
- **编码规范**：Vue 3 + TypeScript strict + ESLint/Prettier
- **性能**：懒加载、分页、包体积与渲染要点

## 快速开始

1. 克隆本仓库：

```bash
git clone https://github.com/wpuing/basic-application.git
```

2. 按端拷贝约束文件：

```bash
# 后端
cp agent-app/AGENTS.md /path/to/your-backend/AGENTS.md

# 前端
cp web-app/AGENTS.md /path/to/your-frontend/AGENTS.md
```

3. 替换占位符：

| 占位符 | 含义 | 示例 |
|--------|------|------|
| `[project]` | 后端基包名 | `com.example.order` |
| `[app]` | 前端工程名 | `order-web` |
| `{biz}` | 业务域 | `order`、`user` |
| `{app}` | Maven 模块前缀 | `order` → `order-service` |

4. 在 Cursor / Copilot 等工具中打开目标工程，Agent 会优先遵循 `AGENTS.md`。

## 推荐技术栈

| 端 | 选型 |
|----|------|
| 后端 | Java 17+、Spring Boot、MyBatis-Plus、MySQL 8、Redis；DDD 四层 |
| 前端 | Vue 3、TypeScript、Vite、Vue Router、Pinia、Axios |
| 规范 | 后端：阿里手册黄山版 + p3c；前端：Vue 风格指南 + ESLint |

## 新功能开发清单（摘要）

**后端：** entity → Repository → Dsvc → Mapper/Impl → Asvc → Ctl + VO  

**前端：** api 类型与方法 →（可选 store）→ views + 域组件 → 路由/权限 → composable

## 贡献

欢迎通过 Issue / PR 补充规约。保持各端 `AGENTS.md` 简洁、可执行；后端模板勿混入前端细节，前端模板勿混入服务端实现细节。

## License

未指定许可证时，默认仅供内部或个人学习使用。上传公开仓库前请自行补充 `LICENSE`（如 MIT / Apache-2.0）。
