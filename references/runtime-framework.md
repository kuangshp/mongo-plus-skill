# 运行时、框架与请求校验

本资料独立提供连接、校验和 HTTP 接入规则，不要求安装 Gin 或 go-zero 相关技能。业务项目使用哪个框架才读取对应部分，不因为技能列出两种框架就引入另一种依赖。

## 1. 生命周期与职责

启动顺序：加载项目配置 → Connect/Ping → 注册生成集合规则及需要的插件 → 注册默认连接 → 初始化校验器 → 构造控制器/接入路由 → 启动 HTTP。优雅停止时先停止接收请求，再关闭 DAL、拥有的 Client 及其他资源。Schema 同步不放每个 HTTP 实例启动入口。

下面是可合并到项目已有初始化包的函数，示例 module 为 example.com/service；实际使用按 go.mod 替换路径。cfg 来自项目配置，Timeout 的单位必须正确，不硬编码数据库 URI。

```go
package bootstrap

import (
    "context"
    "time"

    mongoplus "github.com/kuangshp/mongo-plus"
    "example.com/service/query/repository"
)

func OpenMongo(ctx context.Context, cfg mongoplus.Config) (*mongoplus.Database, error) { // 连接并核验数据库，注册生成集合规则及默认数据源。
    db, err := mongoplus.Connect(ctx, cfg)
    if err != nil {
        return nil, err
    }
    if err := repository.RegisterGeneratedCollections(db); err != nil {
        closeCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()
        _ = db.Close(closeCtx)
        return nil, err
    }
    // 项目若需要插件，在 Use/构造 Repository 前配置并处理错误。
    mongoplus.Use(db)
    return db, nil
}

func CloseMongo(db *mongoplus.Database) error { // 在服务退出时使用独立清理上下文关闭共享数据库连接。
    if db == nil {
        return nil
    }
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    return db.Close(ctx)
}
```

用已有官方驱动 Client 时，可通过 NewDatabase(nativeDB) 包装，不重复 Connect；关闭同一个 Client 的所有者保持唯一。多个数据库的绑定由用户指定，不按请求调用 Use 切库。构造 Repository 失败可能直接 panic，应在接收请求前保证默认库有效，不能捕获后当成正常业务继续。

## 2. Gin 的文件与调用链

- 模块沿现有端/领域组织，如 internal/api/backend/system/user/{user.go,dto/,vo/,logic/}。HTTP 控制器定义 Go interface，实现结构体的方法接收 *gin.Context，构造函数返回接口；复杂编排放现有模块 Logic，不造空 Service。
- 请求/响应类型分别放 dto/vo，不混进生成 Model 或控制器；DTO 可使用公共生成器产物再裁剪。返回 VO 不直接暴露实体中的密码、租户或内部状态。
- 模块路由放 router/<端>/<领域>/<module>.go，领域与端逐级汇总到初始化入口；路由只绑定路径/方法/中间件。沿用项目 URL、鉴权与统一响应，不将模块自动设为公开。
- 调用链为 ShouldBindJSON/ShouldBindQuery → 业务校验/Logic → Repository → DTO/VO 映射 → 项目统一响应。

### 校验器在启动时初始化

Gin 的 ShouldBind 已执行 binding 标签，配置它现有的引擎，不用同一个请求再调用 validation.Validate。以下单独保存为初始化包函数；只调用一次，注册失败终止启动：

```go
package requestvalidation

import (
    "fmt"
    "github.com/gin-gonic/gin/binding"
    "github.com/go-playground/validator/v10"
    "github.com/kuangshp/mongo-plus/validation"
)

func Init() (*validation.Validator, error) { // 为 Gin 现有校验引擎注册公共规则与错误翻译。
    if binding.Validator == nil {
        return nil, fmt.Errorf("Gin 参数校验器已禁用")
    }
    engine, ok := binding.Validator.Engine().(*validator.Validate)
    if !ok {
        return nil, fmt.Errorf("不支持的 Gin 校验引擎")
    }
    return validation.Configure(engine)
}
```

项目自定义规则使用 validation.WithRules 合并到同一次初始化，保留原规则及翻译。内置 dateTime 格式是 2006-01-02 15:04:05，objectId 校验 24 位十六进制；业务解析仍处理错误。可选时间用 omitempty,dateTime，不可用整数时间戳替代。

### DTO 与控制器示例

以下请求示例单独放 dto/user_request.go；Status 使用指针使合法 0 与缺失可区分：

```go
package dto

type CreateUserReq struct {
    Username  string `json:"username" binding:"required,min=2" validate:"required,min=2"` // 用户名，至少两个字符。
    Status    *int64 `json:"status" binding:"required,oneof=0 1 2" validate:"required,oneof=0 1 2"` // 用户状态：0、1 或 2；指针用于区分缺失和合法零值。
    StartTime string `json:"startTime" binding:"omitempty,dateTime" validate:"omitempty,dateTime"` // 可选开始时间，格式为 2006-01-02 15:04:05。
}
```

示例只创建用户，StartTime 用于演示可选时间校验，不映射到未定义的实体字段。控制器文件如下，假定 Schema 的 ID 是默认 ObjectID；字段与 imports 用实际模块路径。简单例子的业务映射留在控制器方法，复杂规则移入模块 Logic，均不能放 Repository：

```go
package user

import (
    "net/http"
    "github.com/gin-gonic/gin"
    "github.com/kuangshp/mongo-plus/validation"
    "go.mongodb.org/mongo-driver/v2/mongo"
    "example.com/service/internal/dto"
    "example.com/service/query/model"
    "example.com/service/query/repository"
)

type IUser interface {
    Create(*gin.Context) // 校验请求并创建用户，返回接口响应。
}

type user struct {
    userRepository repository.UserRepository // 用户数据访问接口，复用生成方法。
    requestValidator *validation.Validator // 启动时配置的请求校验器及错误翻译实例。
}

var _ IUser = (*user)(nil)

func NewUser(requestValidator *validation.Validator) IUser { // 创建用户控制器并初始化用户仓储和校验依赖。
    return &user{
        userRepository: repository.NewUserRepository(),
        requestValidator: requestValidator,
    }
}

func (u *user) Create(ctx *gin.Context) { // 校验用户请求、调用仓储创建用户并返回约定响应。
    var req dto.CreateUserReq
    if err := ctx.ShouldBindJSON(&req); err != nil {
        ctx.JSON(http.StatusBadRequest, gin.H{"message": u.requestValidator.FirstMessage(err)})
        return
    }
    entity := &model.UserEntity{Username: req.Username, Status: *req.Status}
    if err := u.userRepository.Create(ctx.Request.Context(), entity); err != nil {
        if mongo.IsDuplicateKeyError(err) {
            ctx.JSON(http.StatusConflict, gin.H{"message": "用户名已存在"})
            return
        }
        ctx.JSON(http.StatusInternalServerError, gin.H{"message": "保存失败"})
        return
    }
    ctx.JSON(http.StatusOK, gin.H{"id": entity.ID.Hex()})
}
```

启动成功取得非 nil requestValidator 后构造控制器，再由模块注册函数绑定 `group.POST("/users", controller.Create)`；group 来自项目路由层，域/端汇总必须实际调用它。示例 gin.H 为最小展示，已有项目替换为现有响应/VO，避免新增第二套错误或响应协议。

## 3. go-zero 的协议、Handler 与 Logic

目录沿现有 .api、internal/handler、internal/logic、internal/types、internal/svc、etc 布局。ServiceContext 保存配置、连接池和共享资源；Repository 接口字段仍放实际 Logic，并在 NewXxxLogic 返回字面量中初始化，不转移到 ServiceContext。

### 协议与生成

最小 service.api 示例，Status 指针区分缺失与 0；类型及路由按实际业务裁剪，不因为生成器可用就暴露全套 CRUD：

```text
syntax = "v1"

type CreateUserReq {
    Username string `json:"username" validate:"required,min=2"` // 用户名，至少两个字符。
    Status *int64 `json:"status" validate:"required,oneof=0 1 2"` // 状态：0、1 或 2，指针区分缺失与零值。
    StartTime string `json:"startTime,optional" validate:"omitempty,dateTime"` // 可选开始时间，格式为 2006-01-02 15:04:05。
}

type CreateUserResp {
    Id string `json:"id"` // 用户 ID，默认 ObjectID 的十六进制字符串。
}

service service-api {
    @handler CreateUser
    post /users (CreateUserReq) returns (CreateUserResp)
}
```

数据层仍先运行 generator.go。HTTP 代码用项目已经存在的 goctl 与模板；若采用库提供的内嵌模板，可用公共 RunGoctl(ctx, GoctlRunOptions{}, "api", "go", "--api", "service.api", "--dir", ".", "--style=goZero")，或已安装的 `mongoplus goctl api go --api service.api --dir . --style=goZero`。不要求先装 CLI，也不临时下载/升级 goctl。

goctl 生成 Types、Handler、Logic 和路由；已有文件可能保留不更新，核对签名与实际注册。DTO/VO 生成器和 .api types 是不同职责，按需映射，不强制合成同一个类型，也不重复生成另一套业务层。

### Handler 校验链

生成 Handler 依次：decimaljson.PrepareRequest（仅在项目已有数字兼容需求/模板采用时）→ httpx.Parse → validation.HasRules/Validate → NewXxxLogic → 业务方法 → 统一响应。完整的关键校验片段如下，req 是当前 types 请求值，失败立即返回，不调用业务：

```go
if err := httpx.Parse(r, &req); err != nil {
    httpx.ErrorCtx(r.Context(), w, err)
    return
}
if validation.HasRules(&req) {
    if err := validation.Validate(&req); err != nil {
        httpx.ErrorCtx(r.Context(), w, err)
        return
    }
}
```

导入 github.com/kuangshp/mongo-plus/validation。已有统一解析与错误适配先复用，不能只在 Types 留标签而不调用校验，也不校验两遍。Handler 调用 NewCreateUserLogic(r.Context(),svcCtx,r,w) 时必须与真实构造函数一致；若模板尚未生成 request/w 参数，在已有 Handler/Logic 中成对补齐，不能只改一端。

### Logic 使用形态

构造函数字段示例已完整放在[生成流程](generator-workflow.md)，属于本包内容。CreateUser 的字段换成 `userRepository repository.UserRepository`，赋值为 `repository.NewUserRepository()`；Logger、ctx、svcCtx、Request、W 按该示例保留。业务方法中：

```go
func (l *CreateUserLogic) CreateUser(req *types.CreateUserReq) (*types.CreateUserResp, error) { // 将已校验的请求映射为用户实体，保存后返回用户 ID。
    if req == nil || req.Status == nil {
        return nil, fmt.Errorf("缺少用户状态")
    }
    entity := &model.UserEntity{Username: req.Username, Status: *req.Status}
    if err := l.userRepository.Create(l.ctx, entity); err != nil {
        return nil, err // 在已有业务错误适配处转换重复键等错误，不放进 Repository。
    }
    return &types.CreateUserResp{Id: entity.ID.Hex()}, nil
}
```

imports 从本项目取得 types/model/repository，fmt 使用标准库。项目完整业务校验仍在该方法或已有 Service 中。请求 Context 必须传至 Repository；HTTP 请求/响应字段不应进入数据层。

运行时配置可以保留项目 MongoConfig.Enabled/URI/Database/TimeoutSeconds 等既有结构，将秒数检查后转成 time.Duration 再调用 OpenMongo 或现有初始化。需要数据库的接口不能在 Enabled=false、连接为 nil 时继续工作；生成成功不等于启用了运行时数据库。

## 4. 接入验证

确认完整的配置→连接→默认库→构造函数→校验→Logic→Repository→响应链和路由入口。至少对缺失必填、合法 0、非法日期/ObjectID、未找到、重复键和数据库错误按本次修改验证；真实数据库不可用时区分 HTTP 解析/编译检查与数据库集成检查，不报告混合结论。
