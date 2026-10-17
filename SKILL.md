---
name: mongo-plus-skill
description: 在使用 github.com/kuangshp/mongo-plus 的 Go 项目中，按 Schema JSON、根目录生成入口及实际数据源配置开发接口，规范生成目录、Repository 复用与命名、业务分层、BSON 查询和 Session 事务。用户提及该 Go 库，或项目实际依赖该库时使用；普通 MongoDB 驱动问题、其他同名库及仅使用 gorm-plus 的项目不启用。
---

# mongo-plus 开发指南

按当前项目实际版本完成开发或排查。本目录是完整、独立的技能包：使用者只需安装 mongo-plus-skill，不要求安装、读取或复制 gorm-plus-skill、Gin 技能或其他开发技能。共同开发规范及 MongoDB 专用流程均在本包正文和 references 中定义；不得以缺少其他技能为由跳过要求。用户当前要求及适用的 AGENTS.md 优先。

## 先确认环境

1. 阅读项目说明、工作区改动及目标模块，检查 `go.mod`、`go.work`、`replace`、vendor。必要时用 `go list -m -json github.com/kuangshp/mongo-plus` 定位实际源码，不执行升级。
2. 阅读根目录 `generator.go`、生成配置、目标集合 Schema、现有 Entity/DAO/Repository 和调用方。不能仅凭版本推断旧生成文件的方法集。
3. 确认本次需要纯代码生成，还是集合、校验器、索引同步；两者完成状态分别报告。只读审查及无关工具修改不触发生成或数据库写入。
4. 按任务读取参考资料，不默认加载全部：

| 任务 | 资料 |
| --- | --- |
| 新增接口、集合/字段、生成及覆盖问题 | [生成与接口流程](references/generator-workflow.md) |
| 从零配置、字段类型、嵌套对象、索引与迁移 | [配置与 Schema](references/schema-config.md) |
| 连接生命周期、Gin/go-zero、校验与 HTTP 注册 | [运行时与框架接入](references/runtime-framework.md) |
| CRUD、BSON 类型、范围、分页、聚合与请求校验 | [Repository 与查询](references/repository-query.md) |
| 跨集合业务、事务内读取、缓存与 Context | [事务与上下文](references/transactions-context.md) |
| 租户、操作人、数据权限、敏感字段和 DAL | [插件与 DAL](references/plugins-dal.md) |
| 字段与方法备注、生成注释缺失 | [字段与方法备注](references/code-comments.md) |
| 维护技能、复核版本及验证行为 | [来源与验证](references/verification.md) |

## 强制开发约束

- **先建集合再生成基础代码（强制）**：业务接口开发按 `Schema/根目录 generator.go 配置 → 显式创建或同步并核对真实集合、校验器和索引 → 生成 Entity/DAO/Repository → 业务对象初始化 → 接口实现` 执行。配置来自项目，已有集合不重复创建。公共 `mongoplus.Generate` 配置数据源后会先同步再生成，也可先用 `SyncSchema` 同步并核对。准备入口/Schema 不等于已建集合，禁止先生成基础代码、依赖首次业务写入建集合或手写基础代码绕过。只在明确的纯代码生成、模板验证或离线示例任务中允许无数据源生成，并报告数据库未同步，不能充当完整业务开发流程。不要创建 MySQL 日期 SQL、假设可从 MongoDB 反查生成全部类型，或用 HTTP 生成代替数据层生成。
- **字段与方法备注（强制）**：新增或修改的结构体字段（含嵌入字段、DTO/VO、配置及依赖字段）必须在声明右侧用 `//` 写清含义；接口方法签名右侧、函数/构造函数/方法定义行右侧必须说明用途，定义使用 `func ... { // 方法作用`。多行签名在最后一行说明，不把备注只留在调用处；已有 GoDoc 保留并完善。生成字段和方法的备注通过 SQL/Schema 注释、配置或实际模板补齐后重生成，禁止手改生成文件或为加备注新建 Model/Repository 文件。具体格式见[字段与方法备注](references/code-comments.md)。
- **Model/Entity 目录由生成器独占**：`query/model` 及真实 `ModelPkgPath` 输出目录和子目录中的全部文件禁止手写、手改或手动新建，不以文件后缀、额外方法或扩展文件绕过。实体、字段、类型、标签及注释从 Schema、配置或实际模板修正后重生成；失败不能手写补齐。DAO 及其他自动覆盖产物同样通过生成流程更新。
- **Repository 不手动新建文件**：包括子目录、helper/query/extra 文件。新文件只能由生成器产生；模板能力确实缺失时，仅在对应集合已有非生成文件中新增纯数据访问方法，并更新已有公开接口。禁止手改 `_gen.go` 或复制另一套 Repository。
- **模板能力优先复用**：沿默认接口和实现嵌入检查完整方法集，已有 CRUD、Wrapper、分页、聚合和事务绑定可用就直接调用。禁止为对齐 HTTP 名称或接收 Req/DTO，再定义 `CreateXxx`、`DeleteXxx`、`RenameXxx` 等改名、改参或简单转调的重复方法；保留模板类型、嵌入与构造函数。
- **业务代码禁止放进 Repository**：业务校验、权限/状态判断、计算、业务错误文案及跨 Repository 流程由 Logic/Service 或已有业务层负责。Repository 只返回数据、数量或持久化错误。禁止通过覆盖 CRUD、私有函数或改模板植入业务规则。业务层也不能绕过 Repository 直接写 DAO/驱动查询链。
- **Gin 与 go-zero 使用相同 Repository 形态**：业务结构体声明完整名称的接口字段，如 `purchaseOrderRepository repository.PurchaseOrderRepository`；在该对象 `NewXxx(...)` 返回的结构体字面量中直接初始化。调用使用完整字段名，禁止 `repo`、`poRepo`、`purchaseOrderRepo`、Repository 包别名或另设局部别名；不改为从 ServiceContext/启动容器转取已构造的 Repository。
- **连接参数**：单库已通过 `mongoplus.Use(db)` 设置默认连接且真实签名支持时，可用 `repository.NewPurchaseOrderRepository()`，显式传项目现有连接也允许。多库参数由用户自行填写，不猜库、不填 nil 或添加回退；必要参数未填时明确待完成。构造 Repository 不新建 Client，不在请求中连接数据库。
- **查询使用完整 DAO 名称**：如 `dao.PurchaseOrderEntity.Status`，不定义实体局部别名。普通查询变量、回调参数及聚合输出别名不受此限制。先核对 BSON 类型，不能把 ObjectID 字符串直接当成 ObjectID 条件。
- **分页与范围**：普通分页优先生成的 `FindPage`/`FindPageByWrapper`，聚合投影分页用真实 `ScanByPage` 签名，不自行拼 Count/Skip/Limit。需要直接计算偏移时只用 `mongoplus.DbPage`。双端非零范围用 `BetweenIfNotZero`；合法零边界或允许的单端输入必须显式处理，不能静默漏筛选。
- **请求时间为 string**：添加 `dateTime` 规则并实际接入校验；沿用 Gin `binding` 或 go-zero/普通 Go 的 `validate`。解析、DTO/VO 映射和业务异常转换放在业务层或已有 Mapper。
- **事务按 MongoDB 语义**：业务层组织 `TransactionCtx` 与 `WithTx(txCtx).普通方法(...)`，读取和写入共享事务；不从 gorm-plus 复制 `dao.Use`、事务 DAO 或 `CreateTx` 签名。不得通过把业务规则放回 Repository 或把校验移出事务来解决分层问题。

库提供 `GenerateModels` 支持已有手写 Entity，但新项目默认走 JSON Schema 生成路线，不以该 API 绕过 Model 禁写约束。已有项目采用手写来源时先识别来源边界，不擅自迁移或维护第二份 JSON，详见生成流程。

## 完整接口实施顺序

确认模块与依赖 → 明确请求/响应及集合 → 准备根目录入口与 Schema/配置 → 先创建/同步并核对真实集合及结构，再生成基础代码（明确的纯离线生成任务除外） → 核对生成清单 → 初始化连接和必要插件 → 在业务对象构造函数初始化 Repository → 实现业务校验、调用与 DTO/VO 映射 → 注册 HTTP 路由及校验链 → 编译和相关验证。修改已有接口时复用已有文件及入口，不重复造配置、数据层或连接。

Model/DAO/Repository 文件以生成配置为准；DTO/VO 放实际接口模块或配置的契约目录，业务层放模块 Logic/Service，路由只绑定 HTTP，初始化只装配共享资源。只创建需求所需接口，不因模板支持全部 CRUD 就全部暴露。Gin 与 go-zero 的具体布局及完整调用示例见运行时资料。

Skill 不授权生产库操作、破坏性迁移、全量生成、升级库、安装 CLI、推送或部署。结构同步与代码生成按本次任务已有授权及环境执行；普通开发不额外开启缓存或搭建基础设施。

## 完成检查

- 本次涉及的字段声明、接口方法及函数/方法定义右侧均有准确备注；生成产物的备注来自可复现的输入或模板，未手改输出。
- 生成入口、Schema 来源、配置及目标集合明确；业务开发中真实集合/结构同步核对先于基础代码生成，最后才做业务实现。数据库同步、文件生成、HTTP 生成分别有实际结果，纯离线任务不冒充完整业务流程，未做的步骤如实标记。
- Model/Entity 无手写改动，Repository 无手动新增文件，无重复 CRUD 或业务规则，新增数据方法有明确能力缺口。
- 完整字段名和构造函数内初始化保持一致；默认连接有效，多库参数未猜填。
- BSON ID、更新结果、未找到及零值语义处理正确；分页、范围与请求时间校验符合规则。
- 事务校验和读写保持同一 Session，Context、租户、权限、缓存机制未绕过。
- 请求 DTO/响应 VO 与实体分离；校验器已接入且非法输入确实被拒绝；路由完整注册，未改变已有认证、URL 或响应契约。
- 技能内所有相对引用都能在单独复制的本目录中解析；执行不依赖其他技能或作者本机库路径。
- 执行项目所需格式、编译和相关测试，不对无关仓库全量格式化；交付列出实际验证及未验证内容。
