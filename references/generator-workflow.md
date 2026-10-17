# 生成与接口流程

完整 YAML、嵌套 Schema 和索引示例见本包[配置与 Schema](schema-config.md)；连接、框架初始化及请求校验见[运行时与框架接入](runtime-framework.md)。无需读取其他技能。

## 1. 准备来源与入口

- 根目录是目标服务所属 Go module 根目录。复用已有 `generator.go`，缺失才创建；`Package` 从 `go.mod` 取得，输出路径复用项目约定，无既定布局时可用 `query/model`、`query/dao`、`query/repository`。
- 默认每个集合一个 Schema JSON，直接放真实 `SchemaDir` 中；基线只加载目录直接包含的 `.json`，不递归读取。它是稳定的集合定义，不套用日期 SQL 或年月日子目录。
- JSON 是 mongo-plus 实体定义格式，不是原生 `$jsonSchema`。字段用 `name`/`type`，附加约束放 `rules`；自动 ID、时间、租户及软删除字段不要重复声明。集合名、主键类型和索引按业务确定，不猜复数形式。

最小 `schema/user.json`：

```json
{
  "comment": "用户",
  "fields": [
    {"name": "username", "type": "string", "required": true, "unique": true},
    {"name": "status", "type": "int64"}
  ]
}
```

先复用项目入口；缺失时可用下例读取显式指定的配置，不以示例配置作为连接失败的回退：

```go
//go:build ignore

package main

import (
    "log"
    "os"

    mongoplus "github.com/kuangshp/mongo-plus"
)

func main() {
    if len(os.Args) != 2 {
        log.Fatal("用法: go run generator.go <生成配置文件>")
    }
    cfg, err := mongoplus.LoadGeneratorConfig(os.Args[1])
    if err != nil {
        log.Fatal(err)
    }
    if err := mongoplus.Generate(cfg); err != nil {
        log.Fatal(err)
    }
}
```

配置可以用项目已有 YAML，也可在 Go 中从项目配置类型映射 `GeneratorConfig`，不强制另建连接配置。基线 Go 相对路径按执行工作目录解析，YAML 路径按配置文件目录解析；`SchemaDir` 的 YAML 键为 `schemaDir`，其余键以 `schema.Config` 为准。模板为库内嵌资源，不假定存在 `TemplatePath` 等覆盖参数；改库模板需确属本次范围，不悄悄升级依赖。

## 2. 区分生成与数据库同步

| 入口/配置 | 基线行为 |
| --- | --- |
| 公共 `mongoplus.Generate(cfg)` 未提供 MongoDB 配置 | 从 JSON 生成文件，不连接数据库 |
| 公共 `mongoplus.Generate(cfg)` 配置了 MongoDB | 先同步集合、校验器和索引，失败停止生成 |
| 底层 `generator.Generate(cfg)` | 只生成文件，成功不代表数据库已同步 |
| `mongoplus.SyncSchema(ctx, cfg)` | 只同步，不生成，需要完整数据源 |

- 连接来自本次项目配置或其环境变量映射，不硬编码示例库。已声明数据源却缺少 URI/database 是错误，不得移除配置偷偷降级为纯生成。
- 需要同步时确认环境、目标集合及相同 Schema/配置，可用已安装 CLI 的 `schema plan` 看差异，再执行项目同步。不因技能提到 CLI 就安装，也不只为生成代码启动 MongoDB。
- 同步不是整体原子事务，不删除旧字段、集合或索引，不自动回填历史文档。重命名、类型转换、数据回填和索引替换走显式迁移；可核对项目已有 `RunMigrations`，不伪造 SQL。严格校验规则变化需检查旧文档兼容性。
- 同步失败先查真实状态，保留可继续的配置与 Schema，不声称全部完成。离线生成时明确数据库未核对，不当成错误强行写库。

## 3. 运行与产物检查

从模块根目录运行真实入口，例如 `go run generator.go <项目配置>`。基线读取整个 SchemaDir，不编造单集合参数，不删除其他 Schema 来缩小范围；涉及多个集合时核对输出清单及任务范围。

| 产物 | 基线行为 |
| --- | --- |
| Model `*_entity.go`、DAO `*_fields_gen.go` | 生成并更新，禁止手工编辑 |
| Repository `*_gen.go`、`collections_gen.go` | 生成并更新，禁止手工编辑 |
| Repository 同名非生成 `.go` | 首次创建，已有保留，仅允许缺失的纯数据方法扩展 |
| 可选 DTO `*_dto.go`、VO `*_vo.go` | 配置输出路径才首次创建，已有保留，可按接口契约修改 |

Model 输出目录全部由生成器独占，不仅看后缀；禁止手动新增实体、方法、标签、注释或 `extra.go`。不要伪造生成注释骗过覆盖检查；同名非生成文件冲突先查来源，不强制覆盖。实体删除/重命名不自动清理旧产物，应先核对引用，再按项目生成维护流程整理旧文件，不误删自定义实现。

Schema 修改后 DTO/VO、已有扩展与 HTTP 文件不一定更新，应检查契约与映射。HTTP 生成走项目已有 goctl/框架流程，不将 `.api` 或 go-zero 布局强加给 Gin。

已有项目若以 `GenerateModels` 注册手写 Entity：识别真实来源位置，它不覆盖原实体。保持既有来源，不额外复制 JSON；不授权在生成 Model 目录手写，也不擅自迁移全项目。确需编辑手写来源时先对照用户约束解决冲突，不能以库支持为由绕过。

## 4. 初始化与接口接入

- 启动时连接，拥有 Client 的入口负责关闭；无参 Repository 构造前设置 `mongoplus.Use(db)`。`RegisterGeneratedCollections(db)` 按实际需要在服务启动、创建 DAL 前注册各数据源的集合规则；这是运行时注册，不是 Schema 同步。
- Repository 在业务对象构造函数内初始化，不要求把实例放进 ServiceContext。go-zero 形态如下，imports 用项目真实路径：

```go
type ListOrdersLogic struct {
    logx.Logger
    ctx     context.Context
    svcCtx  *svc.ServiceContext
    Request *http.Request
    W       http.ResponseWriter

    purchaseOrderRepository repository.PurchaseOrderRepository
}

func NewListOrdersLogic(ctx context.Context, svcCtx *svc.ServiceContext, request *http.Request, w http.ResponseWriter) *ListOrdersLogic {
    return &ListOrdersLogic{
        Logger:  logx.WithContext(ctx),
        ctx:     ctx,
        svcCtx:  svcCtx,
        Request: request,
        W:       w,

        purchaseOrderRepository: repository.NewPurchaseOrderRepository(),
    }
}
```

- 单库无参依赖默认连接及真实可变参数签名，显式传现有数据库句柄也允许；多库由用户填参，不猜字段名、传 nil 或按请求切全局默认库。缺参数时说明待填写，不宣称编译通过。
- Gin 在已有控制器实现/模块 Logic 中保持同样完整接口字段与构造函数初始化，保留控制器接口、DTO/VO 及路由布局，不引入 go-zero 的 ServiceContext/logx 等专用字段。
- 业务层校验、映射后调用模板方法。末级分类、必选属性、存量商品等规则都留在业务层，不覆写 Create。见[查询规则](repository-query.md)及[事务规则](transactions-context.md)。
