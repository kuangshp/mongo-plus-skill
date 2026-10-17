# 插件、数据权限与 DAL

本资料按需使用，不因为加载技能就给项目加租户、缓存、加密或 DAL。所有插件在启动时按当前数据源注册；现有功能继续保留，不重新实现一套过滤/填充机制。

## 插件接入顺序

1. 核对 Schema 中的租户、审计字段、索引及 BSON 类型，先按生成流程更新；不能手改 Entity。
2. 在已有连接上注册插件，处理失败；之后构造 Repository/启动服务，不在请求中重复注册。
3. 鉴权中间件从服务端已验证身份取得租户、操作人和权限范围，写入标准请求 Context；Gin 应使用 ctx.Request = ctx.Request.WithContext(requestCtx)，go-zero 传 r.WithContext(requestCtx)。不能只 c.Set 后假设库读得到，也不信任前端自报租户。
4. 业务方法原样传 Context；事务回调与绑定仓储保留同一身份。分别验证不同租户/权限的数据隔离及缓存结果。

## 已有公共能力

| 需求 | 基线 API 与边界 |
| --- | --- |
| 集合租户字段 | 生成配置 tenant/CollectionConfig.TenantField，结合 WithTenant 或 WithTenantID 写入身份 |
| 多租户插件 | RegisterTenant(db,TenantConfig[T])；字段、集合排除与 GetTenantID 按项目定义 |
| 操作人自动填充 | RegisterAutoFill、FieldConfig、OperatorGetter[T]；请求身份用 WithActor/WithOperator |
| 数据权限 | RegisterDataPermission、WithDataPermission；默认缺失注入报错，不以 AllowMissing 逃避鉴权 |
| 敏感字段 | RegisterSensitive；查询优先仓储 SensitiveEq，返回展示模式按实际请求处理 |
| 主从/多源 | 复用 DS.Register/DS.Auto 等已有注册；用户确定数据源，业务开发不自动选库或改路由策略 |

以下是租户插件的初始化函数，仅在实际需要且 Schema/运行规则一致时使用，不强制向普通服务追加租户：

```go
package bootstrap

import mongoplus "github.com/kuangshp/mongo-plus"

func ConfigureTenant(db *mongoplus.Database) error { // 为指定数据源注册 int64 租户字段与上下文取值规则。
    return mongoplus.RegisterTenant(db, mongoplus.TenantConfig[int64]{
        TenantField: "tenant_id",
        GetTenantID: mongoplus.TenantIDFromCtx[int64],
    })
}
```

鉴权完成后可用 `requestCtx := mongoplus.WithTenantID(r.Context(), verifiedTenantID)`，再用 `mongoplus.WithActor(requestCtx, verifiedActorID)`。verified 值来自既有认证结果，并处理缺失或类型不符；示例不提供匿名/默认租户回退。请求 ctx 不得替换为 Background。技术清理 Context 不用于业务查询。

自动填充字段须存在于 Schema，配置 FieldConfig{Name,Getter,OnCreate,OnUpdate}；租户或操作者的集合类型不同，Getter 也必须匹配。数据权限回调基线类型为 `func(collection string) (bson.D,error)`，它返回服务端可信过滤；业务权限决策来自业务/鉴权层，Repository 负责应用既定约束。

禁止为绕过失败启用 SkipTenant、SkipDataPermission、AllowGlobalOperation、全集合 AllowAll 或排除当前集合。受控跨租户/管理操作只能依照明确需求及既有授权实现。

## DAL 文件组织与调用

普通操作先用模板 Repository；确有复杂聚合、批量或官方扩展需求再用 DAL。DAL JSON/Extended JSON 表达数据操作，不执行 SQL/JavaScript，不把业务判断搬进查询文件。

查询文件与 embed 加载器放项目已有 query 资源目录，不能在 Repository/Model 目录新建文件。可用结构：query/files.go、query/user/page.json、query/user/count_page.json；具体位置以项目约定为准。

- files.go 通过 go:embed 引入 JSON，启动时 `NewDal(db, NewEmbedLoader(files))` 并处理错误；Preload 只保证能读取，不代表语义及绑定已通过。DAL Close 只关闭自身清理任务，不关闭共享 Client。
- 纯数据扩展方法写在对应已有 customerXRepository 文件中，使用执行器已有 WithDAL(ctx,dal,fn)；回调里的 DAL 调用必须传 queryCtx，以保留数据源、事务与作用域，不换回原 ctx。
- 不照搬库演示中另建 Repository 类型或手写 Entity 的组织方式，本技能仍禁止第二套实体/仓储。需要 DAL 依赖时使用已有装配位置和字段，WithTx 副本也须保留依赖。

分页数据文件示例；collection 用真实集合名，参数由服务端构造：

```json
{
  "collection": "users",
  "op": "find",
  "filter": {"status": "@status"},
  "sort": {"_id": 1},
  "limit": "@limit",
  "skip": "@offset"
}
```

对应 count_page.json：

```json
{"collection":"users","op":"count","filter":{"status":"@status"}}
```

在已有 Repository 方法的 WithDAL 回调中，参数由 `offset,limit := mongoplus.DbPage(pageNumber,pageSize)` 取得，再调用 DALQueryPageNamed[VO](queryCtx,"user/page.json",params)。命名 count 文件与列表采用相同过滤/聚合语义；展开后的行数不能误作主文档数量。

| DAL 能力 | 注意事项 |
| --- | --- |
| DALQuery/DALQueryNamed | 列表，值按真实 BSON 类型绑定，不字符串拼接 |
| DALQueryOne/DALQueryOneNamed | 基线无记录返回 nil,nil，与 Repository ErrNotFound 不同 |
| DALQueryPage/DALQueryPageNamed | 复用 count_文件与 DbPage，计数及列表不天然同一快照 |
| DALExec/DALExecAffected | 写入及影响结果，检查部分成功和额外缓存失效 |
| DALBulkWrite | ordered 与 unordered 都不自动成为整批事务，不盲目重放 |
| DALWalk | 流式消费，回调错误结束读取，不一次载入全部文档 |
| DALWithTx | 回调为 txCtx，业务层负责跨操作流程；若已在仓储事务中，用其 WithDAL 继承，不能重复开嵌套事务 |
| DALWatch | 变更流需支持的部署；消费幂等及进度持久化在业务层，不放事务中 |

命名参数只替换完整值，ObjectID/time.Time/Decimal128 保留类型；文件路径和操作符由服务器固定，不接受前端指定任意文件、管道或操作符。

## 过滤、缓存与日志

基线 WithDAL 已沿用选定数据源的租户/权限/软删除机制，但原生复杂聚合、关联子集合、未注册集合和不同版本要核对实际覆盖范围；不能根据全局文档一句话推断所有分支都安全，也不能假设 DAL 永远不带过滤而重复注入矛盾条件。必要时在文件显式约束，按真实执行结果验证。

写入正常失效目标集合，其他受影响集合按当前机制登记 invalidates；跨实例缓存 Namespace 与提交后失效见事务资料。原生命令与 $out/$merge 等必须走真实写入口，不能用读入口隐藏写入。

命令日志沿用项目已配置监控器与脱敏，日志级别和超时从配置读取；不在业务函数临时加一套日志中间件。Replayable 模式会暴露真实值，只在用户明确要求且受控环境开启，不将包含凭据、敏感数据或未脱敏参数的日志写进技能或交付报告。
