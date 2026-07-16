# AGENTS.md

> Java 后端编码约束。包结构参照 DDD 项目 `zhcz-zxzj-app`；编码规范对齐《阿里巴巴 Java 开发手册（黄山版）》。  
> 占位符 `[project]` / `{biz}` 按实际项目替换。

## 1. 项目概览

- **技术栈**：Spring Boot + MyBatis-Plus + MySQL 8.0 + Redis（+ MQ / Feign 按需）
- **架构**：DDD 四层 + 按业务域垂直切分；模块化单体，可拆微服务
- **Maven 模块（推荐）**：
  | 模块 | 职责 |
  |------|------|
  | `{app}-service` | 可运行服务：interfaces / application / domain / infrastructure |
  | `{app}-common` | 枚举、常量、工具、跨层 VO |
  | `{app}-client` | 对外 Feign SDK（供他服务调用） |
- **前端**：前后端分离；本文件只约束后端

## 2. 包结构（DDD）

基包：`com.[project]`。先按层，再按业务域 `{biz}`。

```text
com.[project]
├── [Project]Application.java          # 启动类
│
├── interfaces/                        # 接口层（入站适配）
│   ├── config/                        # Web / DB 等接口侧配置
│   └── {biz}/
│       ├── controller/                # *Ctl（REST，可继承 BaseController）
│       ├── vo/                        # *VO / *QueryVO / *ExcelVO
│       └── wrapper/                   # Entity ↔ VO 转换
│
├── application/                       # 应用层（用例编排）
│   ├── service/{biz}/                 # I*Asvc / *AsvcImpl
│   └── util/                          # 应用侧工具（可选）
│
├── domain/                            # 领域层（核心业务）
│   └── {biz}/
│       ├── entity/                    # 领域实体（可 @TableName）
│       ├── repositories/              # I*Repository 仓储接口
│       ├── service/                   # I*Dsvc / *DsvcImpl 领域服务
│       ├── validate/                  # 领域校验/组装（按需）
│       └── dto/                       # 领域内 DTO（按需）
│
├── infrastructure/                    # 基础设施层（出站适配）
│   └── db/
│       ├── mapper/{biz}/              # *Mapper.java + *Mapper.xml（可同目录）
│       └── repositories/{biz}/        # *RepositoryImpl
│
└── mq/                                # MQ 消费者等横切入口（按需，再调 Asvc）
```

**common 模块**（独立 artifact）：`common.constant` / `enums` / `utils` / `vo`  
**client 模块**（独立 artifact）：`feign.I*FeignService` + Fallback + 请求 VO

### 调用链

```text
*Ctl → I*Asvc(*AsvcImpl) → I*Dsvc(*DsvcImpl) → I*Repository(*RepositoryImpl) → *Mapper
```

### 层职责与依赖

```text
interfaces → application → domain ← infrastructure
                ↓            ↑
              common ←───────┘
```

| 层 | 允许 | 禁止 |
|----|------|------|
| **interfaces** | 入参校验、VO 转换、调 Asvc | 直接调 Mapper / RepositoryImpl |
| **application** | 用例编排、`@Transactional`、组合多个 Dsvc | 塞复杂领域规则（下沉 Dsvc） |
| **domain** | 实体、领域服务、仓储接口、业务校验 | 依赖 Web、Mapper 实现类 |
| **infrastructure** | DB、外部 Client、仓储实现 | 被 interfaces 直接调用 |

### 命名后缀

| 后缀 | 层 | 示例 |
|------|----|------|
| `Ctl` | interfaces | `OrderCtl` |
| `VO` / `QueryVO` | interfaces | `OrderVO`、`OrderQueryVO` |
| `Wrapper` | interfaces | `OrderWrapper` |
| `I*Asvc` / `*AsvcImpl` | application | `IOrderAsvc` |
| `I*Dsvc` / `*DsvcImpl` | domain | `IOrderDsvc` |
| `I*Repository` / `*RepositoryImpl` | domain / infra | `IOrderRepository` |
| `*Mapper` | infrastructure | `OrderMapper` |

### 新功能清单

1. `domain.{biz}.entity` 实体  
2. `domain.{biz}.repositories` → `I*Repository`  
3. `domain.{biz}.service` → `I*Dsvc` / `*DsvcImpl`  
4. `infrastructure.db` → Mapper + `*RepositoryImpl`  
5. `application.service.{biz}` → `I*Asvc` / `*AsvcImpl`  
6. `interfaces.{biz}` → `*Ctl` + VO + Wrapper  

## 3. 外部服务 / 高延迟调用

适用于：第三方 OpenAPI、Feign、SSE、重计算、LLM。

| 项 | 普通请求 | 流式 SSE |
|----|----------|----------|
| 线程池 | 核心 20 / 最大 50 / 队列 100 / CallerRunsPolicy | 30 / 80 / 50 / AbortPolicy |
| 超时 | connect 5s · read 120s · write 30s | connect 5s · read 0 · write 30s |
| 重试 | ≤3（500ms 起、×2、上限 10s）；仅网络异常与 5xx | |
| 熔断 | 窗口失败率 >50% 或慢调用占比过高；开路后半开探测；可配置 fallback | |

## 4. 数据库

### 通用字段

```sql
id          BIGINT       NOT NULL AUTO_INCREMENT,  -- 禁用 UUID 作主键
create_time DATETIME(3)  NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
update_time DATETIME(3)  NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
deleted     TINYINT(1)   NOT NULL DEFAULT 0,
PRIMARY KEY (id)
-- utf8mb4 / utf8mb4_unicode_ci；金额 DECIMAL(19,4)；布尔 TINYINT(1)
```

### 索引与查询

- 表名、字段名小写 + 下划线；表建议 `t_` 前缀
- 低区分度字段不单独建索引；组合索引最左前缀；条件含 `deleted` 则索引带上
- 每表索引 ≤5；禁止 `select *`；禁止循环内单条 SQL（改批量）
- 禁止深分页；优先游标分页

```sql
SELECT id, name, status FROM t_order
WHERE status = ? AND create_time < ? AND id < ?
ORDER BY create_time DESC, id DESC
LIMIT 20;
```

### 索引治理

- 开发：慢 SQL EXPLAIN，`type=ALL` 告警  
- CI：关键查询全表扫描则失败  
- 生产：关注 `sum_no_index_used > 0`

## 5. 编码规范

> 以《阿里巴巴 Java 开发手册（黄山版）》为准；本项目 DDD 后缀（`Ctl`/`Asvc`/`Dsvc`）优先于手册中的通用 Service 命名习惯。  
> IDE 建议安装 **Alibaba Java Coding Guidelines（p3c）** 插件。

### 5.1 命名风格【强制要点】

- 命名不以 `_` / `$` 开头或结尾；禁止拼音英文混用、禁止中文命名
- 类名 UpperCamelCase（例外：DO/DTO/VO/BO 等后缀）
- 方法/参数/成员/局部变量 lowerCamelCase
- 常量全部大写，单词间下划线，语义完整（如 `MAX_STOCK_COUNT`）
- 包名全小写、单数；点分隔词为完整英文单词
- 杜绝无意义缩写（如 `AbsClass` → `AbstractClass`）
- 抽象类可用 `Abstract` / `Base` 开头；异常类以 `Exception` 结尾；测试类以 `Test` 结尾
- 枚举类名带 `Enum` 后缀或业务清晰命名；成员全大写+下划线
- POJO 布尔属性**不要**加 `is` 前缀（反序列化坑）；基本类型包装类型按场景选用
- **本项目**：应用服务 `I*Asvc`，领域服务 `I*Dsvc`，Controller `*Ctl`（与通用「接口不加 I」习惯不同，以本仓库为准）

### 5.2 常量与魔法值

- 不允许魔法值（未经定义的常数）直接出现在代码中
- long 赋值用大写 `L`（`2L`，不用 `2l`）
- 按功能归类常量类，禁止一个大而全的常量类

### 5.3 代码格式

- 缩进 4 空格，禁止 Tab
- 单行不宜超过 120 字符；运算符换行时符号放行首
- if / for / while / switch 即使单行也必须加大括号
- 方法体行数建议 ≤80；超过则拆分

### 5.4 OOP

- 覆写方法必须加 `@Override`
- 相等比较用 `equals`；常量或确定非空对象调用 `equals`（`"ok".equals(x)`）或 `Objects.equals`
- 慎用继承，优先组合；构造方法禁止写业务逻辑
- DO/DTO/VO 等不同层对象分离，禁止混用一杆捅到底
- 序列化类新增字段注意兼容；浮点等金禁用 `BigDecimal`；比较用 `compareTo`，不用 `equals`

### 5.5 集合

- 判断空用 `isEmpty()`，不用 `size() == 0`
- `ArrayList` / `HashMap` 指定初始容量，避免扩容抖动
- 不要在 foreach 里 remove/add，用 Iterator 或 `removeIf`
- `Collectors.toMap` 必须处理 value 为 null 与 key 冲突
- 返回集合不要返回 `null`，返回空集合

### 5.6 并发

- 线程池必须用 `ThreadPoolExecutor` 显式创建，禁止 `Executors` 创建（易 OOM）
- `SimpleDateFormat` 线程不安全，用 `DateTimeFormatter` 或 ThreadLocal
- `ThreadLocal` 必须在 finally 中 `remove()`
- 锁粒度最小化；优先无锁/并发工具；加锁顺序一致防死锁
- 单例注意线程安全；volatile 不保证复合操作原子性
- `CompletableFuture` 必须指定业务线程池，禁用 `ForkJoinPool.commonPool()`

### 5.7 控制语句

- 超过 3 层 if-else 用卫语句、策略或状态模式重构
- switch 必须有 default；每个 case 以 break/return 结束或显式注释 fall-through
- 高复杂度条件抽成布尔方法或变量，提升可读性

### 5.8 注释

- 类、公共方法必须 Javadoc（用途、参数、返回、异常）
- 注释改代码同步改；及时删除过时注释
- 枚举字段必须注释含义

### 5.9 异常

- 禁止空 catch、禁止 `e.printStackTrace()`；捕获后记录或抛出
- 业务异常统一 `BizException(ErrorCode)`（或项目约定异常），不用裸 `RuntimeException` 表达业务语义
- 不要用异常做流程控制
- finally 不 `return`、不抛会掩盖原异常的新异常
- 事务方法建议 `@Transactional(rollbackFor = Exception.class)`
- NPE 防御：返回空集合优于 null；入参 `@NonNull` / `@Valid`；包装类型注意拆箱 NPE

### 5.10 日志

- SLF4J + Logback（或项目统一门面），`@Slf4j`；禁用 `System.out` / `e.printStackTrace`
- 占位符：`log.info("orderId={}", id)`，禁止字符串拼接
- 禁止循环内打日志；异常日志带完整堆栈：`log.error("msg", e)`
- 级别：DEBUG 调试 · INFO 关键节点 · WARN 可恢复 · ERROR 需介入
- 生产默认 INFO；敏感信息脱敏；外部调用记耗时/关键参数（脱敏后）

### 5.11 安全【强制要点】

- 用户输入必须校验；防 SQL 注入（参数绑定，禁拼接）
- 防 XSS；页面输出编码；CSRF 按架构处理（前后端分离常见禁用表单 CSRF，改用 Token）
- 密码等禁止明文存储与日志打印；权限校验在服务端完成，不信任前端隐藏字段

### 5.12 单元测试【推荐】

- AIR：Automatic、Independent、Repeatable
- 核心业务单测覆盖率建议 ≥70%（核心域力争更高）
- 用断言验证结果，不止执行不 assert；边界值、空值必测

## 6. 认证与授权

- Spring Security + JWT（或平台统一认证），会话 `STATELESS`
- Payload 含用户标识、角色、签发/过期时间；密钥环境变量注入，禁硬编码
- 方法级鉴权（如 `@PreAuthorize`）；写操作校验数据归属，防越权
- 头：`Authorization: Bearer <token>`；可选 `X-Request-Id`

## 7. API 文档

- Knife4j / SpringDoc：`@Tag`、`@Operation`、`@Parameter`、`@Schema`
- 生产关闭 Swagger UI（如 `knife4j.production=true`）

## 8. 性能关注点

| 优先级 | 风险 | 要求 |
|--------|------|------|
| P0 | 高可用 | 外部调用超时 + 线程池隔离 + 熔断/Fallback |
| P0 | DB | WHERE/JOIN/ORDER BY 有索引；禁循环查库 |
| P0 | 并发 | 分布式写有锁或幂等 |
| P1 | DB | 禁深分页；N+1 → 批量 |
| P1 | 连接池 | 合理 maxPoolSize / connectionTimeout |
| P2 | 内存 | 大结果集流式处理 |
| P2 | 合规 | 日志脱敏 |
