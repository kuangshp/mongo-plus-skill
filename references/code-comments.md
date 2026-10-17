# 字段与方法备注

## 备注位置与内容（强制）

- 本次新增或修改的结构体字段在声明右侧写 `//` 备注，包含嵌入字段、请求 DTO、响应 VO、配置、Logic/Service 依赖及可编辑扩展中的字段。必要时注明单位、格式、枚举含义、可选性或生命周期，不只翻译标识符。
- 接口方法在签名右侧说明作用；函数、构造函数和接收者方法定义在带 `{` 的定义行右侧说明作用，例如 `func ... { // 初始化订单查询逻辑`。多行签名在最后一行右侧写，不在签名中间插入行注释截断代码。
- 说明该方法做什么、针对什么对象；必要时补充返回内容、重要副作用或事务要求。已有定义前的 GoDoc 保留并维护；只有调用处、接口处或文件顶部有备注，不能代替实现定义处的用途备注。
- 不要求为每个普通调用或局部临时变量重复加注释，也不为补备注改名、改签名或改变业务行为。只按本次范围维护，不无关重写全项目。

## 现有业务对象中的位置示例

以下演示合并到现有定义中的注释位置，不要求新建结构体、接口或文件；类型及导入沿用项目实际定义：

```go
type ListOrdersLogic struct {
    logx.Logger // 当前请求的日志记录器，携带请求上下文。
    ctx context.Context // 当前请求上下文，传递取消信号和身份信息。
    svcCtx *svc.ServiceContext // 服务共享配置和已初始化的资源。
    Request *http.Request // 当前 HTTP 请求，用于读取请求元数据。
    W http.ResponseWriter // 当前 HTTP 响应输出，供实际需要的响应操作使用。
    purchaseOrderRepository repository.PurchaseOrderRepository // 采购订单数据访问接口，复用模板能力。
}

func NewListOrdersLogic(ctx context.Context, svcCtx *svc.ServiceContext, request *http.Request, w http.ResponseWriter) *ListOrdersLogic { // 创建订单列表逻辑并初始化日志、请求上下文及订单仓储。
    return &ListOrdersLogic{
        Logger: logx.WithContext(ctx),
        ctx: ctx,
        svcCtx: svcCtx,
        Request: request,
        W: w,
        purchaseOrderRepository: repository.NewPurchaseOrderRepository(),
    }
}
```

示例无参构造仅在已注册默认连接且签名支持时使用，多数据库参数仍由用户填写。Gin 控制器接口中的方法使用同样规则：

```go
type IOrderController interface {
    Page(ctx *gin.Context) // 按请求条件分页查询订单并返回接口约定的响应。
}
```

Go-zero .api 中的请求和响应字段也写右侧说明；生成的 Types 由源契约重新生成，不手动补改。JSON 不支持 `//`，MongoDB Schema 的字段说明写 comment 属性，由模板输出 Go 备注；SQL 表/列注释使用数据库支持的语法，不能将 Go 行注释直接搬入 JSON/DDL。

## 生成文件与模板

备注来源是Schema JSON 的实体/字段 comment、生成配置及实际模板。先核对生成器是否传递这些说明，缺少自动字段或方法说明时修正实际模板后重生成；Model/Entity、DAO 和 `_gen.go` 仍禁止手写或手改。若需要修改的库模板不在任务范围且无法完成，报告缺失备注及具体阻碍，不能声称全部符合规则。

方法用途注释只描述真实行为，不借加注释重写模板 CRUD 或加入业务校验。Repository 仍只做数据访问，业务规则仍在业务层。
