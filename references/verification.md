# 来源与验证

## 源码基线

- 模块：github.com/kuangshp/mongo-plus。
- 日期：2026-10-17；提交：`02d55879ce6a36cdac99194c99356842ca498ca4`。
- 创建技能时读取了该提交的本地干净源码；仅作证据基线，不是最低版本，不要求升级或拥有作者机器路径。

实际任务以 go.mod/replace/vendor 解析的源码和已有生成文件为准。库文档展示的启动时建仓储、可覆盖业务方法或手写实体等能力，不覆盖本技能已确定的构造函数初始化、业务分层及生成目录约束。

| 内容 | 优先核对源码 |
| --- | --- |
| 生成及可选数据源 | mongoplus_generator.go、mongoplus_schema.go |
| Schema 格式、路径与类型 | schema/schema.go、internal/settings/config.go |
| 输出及覆盖策略 | generator/generate.go、generator/transport.go |
| 默认接口、构造函数、WithTx | generator/template/repository_gen_template.txt、generator/template/repository_template.txt |
| CRUD、返回值与分页 | repository.go、config.go |
| 条件和聚合分页 | query.go、query_compat.go、join.go |
| Session 及提交后失效 | transaction.go、cache.go，并定位当前其他缓存实现 |
| 迁移与校验 | migration/migration.go、validation/、docs/validation.md |

上游固定提交示例：`https://github.com/kuangshp/mongo-plus/blob/02d55879ce6a36cdac99194c99356842ca498ca4/mongoplus_generator.go`。无法取得源码时说明未验证签名，不用其他同名 MongoPlus 库文档代替。

## 维护检查

维护环境有 skill-creator 时可运行其 scripts/quick_validate.py；它是可选维护工具，不是本技能运行依赖。无该工具时检查 SKILL.md 的 name/description、agents 元数据、包内 Markdown 相对链接与示例签名。所有执行必需的规则和参考资料必须位于本目录内，不能让使用者回看其他技能、历史聊天或作者本地文件。结构通过不等于数据库行为通过；在独立临时 module 用同版本验证生成示例，不改库源码或业务数据库。

按本次范围选场景，不要求每次执行全套：

| 请求/场景 | 预期行为 |
| --- | --- |
| 用户明确只需离线 JSON 生成/模板验证 | 根目录入口生成并编译，不连接库，报告未同步，不宣称完整业务接口完成 |
| 开发业务接口但集合尚未创建或同步失败 | 先创建/同步并核对真实集合、校验器与索引，成功后才生成基础代码；缺连接时保留阻碍，不降级离线生成或等业务写入建集合 |
| 已声明数据源但缺 URI/database | 报错，不删配置假装成功 |
| Model 缺字段、需标签/方法 | 改 Schema/配置重生成，不手改或新建 extra |
| 已有手写实体来源 | 识别 GenerateModels，不重复 JSON、不写生成目录或擅自迁移 |
| 缺少联表查询 | 先查现有能力，只在已有非生成 Repository 加纯数据方法 |
| HTTP 创建/删除/改名 | 用默认 CRUD，不再定义同名业务方法 |
| 末级分类、必选属性、状态限制 | Repository 取数据，Logic/Service 判断、产生业务错误，不覆盖 Create |
| Gin/go-zero 初始化 | 完整接口字段在业务对象构造函数初始化，不另设 repo/实体别名 |
| 单库默认连接、多库缺参数 | 无参前确认 Use；多库由用户填，不猜库、不虚报验证 |
| ObjectID 字符串 | 业务层解析，引用字段 BSON 类型匹配 |
| 更新匹配但 ModifiedCount=0 | 不误当未找到，处理结果及错误 |
| 金额 0 到 100、单端时间 | 不让 BetweenIfNotZero 漏筛选，按契约 Gte/Lte 或校验拒绝 |
| 分页、一对多聚合 | 用真实分页签名，检查 total 语义，不手算偏移 |
| 丢弃 WithTx 返回值、跨轮次使用 | 修正绑定，不回退事务外调用 |
| 事务第二次写失败、业务拒绝 | Logic 返回错误，全部回滚，读写同 Session |
| 新字段、接口方法或方法实现缺少右侧备注 | 在字段/签名/定义右侧补充准确含义或用途，多行定义在最后一行标注 |
| 生成的实体字段或方法没有说明 | 从输入或实际模板补全再生成，不手改 Model/DAO/_gen.go，也不另建文件绕过 |
| 只改普通工具函数 | 不引入 Schema、迁移、生成或缓存改造 |

实际验证应记录版本、输入、命令与未验证项，不能把预期当成已测试结论。

## 本次实际验证记录

2026-10-17，使用上述源码提交与 Go 1.26.9，在独立临时 module 中执行，未连接 MongoDB：

- 从本技能提取 generator.go 和 user.json 示例，运行 `go run -mod=mod generator.go mongo-plus.yaml`，生成 Entity、DAO、Repository、DTO、VO 成功。
- 修改 Schema 增加 email 字段后重新生成，Entity/DAO 更新；已有 Repository 非生成文件、DTO/VO 的自定义内容逐字节保留。再次生成输出一致。
- 运行 `go test -mod=mod ./...`，六个包及包含完整字段初始化、分页、更新结果、删除、两种 ScanByPage 和 WithTx 的调用示例编译通过；临时 module 未包含运行时数据库测试。
- 显式空 mongodb 配置被拒绝，已有输出未改变；没有借删除配置降级来伪装同步完成。
- 技能结构、入口元数据及文档相对链接通过检查。

此轮未验证：真实集合/索引同步、MongoDB 事务提交回滚、Session 重试、插件及分布式缓存行为。框架验证的后续结果见下一节。

## 完整独立包补充验证

2026-10-17，将且仅将 mongo-plus-skill 复制到新的临时目录。根据复制包中的 generator.go、YAML、Schema、初始化与框架示例构建项目；未复制或读取任何相邻开发技能。测试依赖实际 mongo-plus 源码及 Gin/go-zero Go 模块，这些是业务项目代码依赖，不是技能依赖。

- 文档中的离线 YAML 与根目录入口成功生成完整数据层；包含 decimal128、嵌套对象、数组、规则和复合/唯一索引的第二个 Schema 也成功生成，产物经 go test 编译通过。
- OpenMongo/CloseMongo、租户注册、Gin 校验初始化、DTO 和接口式控制器示例编译通过；未执行真实连接函数。
- Gin ShouldBindJSON 测试：合法 0 状态和合法日期通过，缺状态、缺用户名和非法日期拒绝；使用配置的 binding 引擎，无第二次校验。
- 用 goctl 1.10.2 及 mongo-plus 公共 RunGoctl 生成 .api 的 Handler/Types/Logic/路由；合并本包的完整 Repository 字段、构造函数初始化及业务方法后，go test ./gozero/... 通过。
- go-zero 的实际 Handler 在缺必填和非法日期时返回 400，未进入 Logic 构造或数据库调用；Types 校验接受合法 0 状态。
- 包内 Markdown 链接全部解析到本目录，无对其他技能的相对引用、作者绝对源码路径或工具依赖声明。

这些是离线生成、编译和请求边界测试，不代表成功执行过数据库 CRUD。真实同步、索引建立、事务回滚重试、插件数据隔离及分布式缓存仍未连接数据库验证。交付时保持这一边界，不将框架编译或非法请求测试称为数据库集成测试。
