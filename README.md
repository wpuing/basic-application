# basic-application

面向 **Java 后端** 的 AI 编码约束模板：通过 `AGENTS.md` 统一 Agent / 开发者在 DDD 分层、数据库、编码与安全方面的约定。

## 仓库结构

```text
basic-application/
├── README.md                 # 本说明
└── agent-app/
    └── AGENTS.md             # Java 后端 AI 编码入口（核心）
```

将 `agent-app/AGENTS.md` 复制到你的后端仓库根目录（或与 Cursor / 其他 Agent 约定的路径），并按实际项目替换占位符即可。

## AGENTS.md 能做什么

- **DDD 包结构**：`interfaces` → `application` → `domain` ← `infrastructure`（参考专项资金类业务项目实践）
- **命名约定**：`Ctl` / `Asvc` / `Dsvc` / `Repository` / `Mapper` 与标准调用链
- **数据库与性能**：通用字段、索引、分页、外部调用超时与熔断
- **编码规范**：对齐《阿里巴巴 Java 开发手册（黄山版）》强制/推荐要点
- **认证与 API 文档**：JWT / Security、Knife4j 生产关闭等摘要

## 快速开始

1. 克隆本仓库：

```bash
git clone https://github.com/wpuing/basic-application.git
```

2. 将约束文件拷入目标后端工程：

```bash
cp agent-app/AGENTS.md /path/to/your-backend/AGENTS.md
```

3. 替换文档中的占位符：

| 占位符 | 含义 | 示例 |
|--------|------|------|
| `[project]` | 基包名 | `com.example.order` |
| `{biz}` | 业务域 | `order`、`user` |
| `{app}` | Maven 模块前缀 | `order` → `order-service` |

4. 在 Cursor / Copilot 等工具中打开目标工程，Agent 会优先遵循 `AGENTS.md`。

## 推荐技术栈（文档约定）

| 类别 | 选型 |
|------|------|
| 运行时 | Java 17+（按团队实际） |
| 框架 | Spring Boot |
| ORM | MyBatis-Plus |
| 存储 | MySQL 8.0、Redis |
| 架构 | DDD 四层 + 业务域垂直切分 |
| 规范 | 阿里巴巴 Java 开发手册（黄山版）+ IDE p3c 插件 |

## 新功能开发清单（摘要）

按 `AGENTS.md` 约定，新增业务能力建议顺序：

1. `domain.{biz}.entity`
2. `domain.{biz}.repositories` → `I*Repository`
3. `domain.{biz}.service` → `I*Dsvc` / `*DsvcImpl`
4. `infrastructure.db` → Mapper + `*RepositoryImpl`
5. `application.service.{biz}` → `I*Asvc` / `*AsvcImpl`
6. `interfaces.{biz}` → `*Ctl` + VO + Wrapper

## 贡献

欢迎通过 Issue / PR 补充规约条目或修正与团队实践不符之处。提交前请保持 `AGENTS.md` 简洁、可执行，避免堆砌与 Harness / 前端无关的内容。

## License

未指定许可证时，默认仅供内部或个人学习使用。上传公开仓库前请自行补充 `LICENSE`（如 MIT / Apache-2.0）。
