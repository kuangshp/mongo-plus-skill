# 事务、上下文与缓存

## 事务组织

跨集合业务流程与校验放 Logic/Service；Repository 只管数据和事务绑定。基线 `mongoplus.TransactionCtx(ctx, db, fn, opts...)` 回调参数是携带 Session 的 context.Context，不是 GORM DAO；不传 dao.Use、不重复定义 CreateTx/UpdateByIdTx。

下例放已有业务方法内，字段已在构造函数初始化，实体、连接及 Context 使用项目真实变量：

```go
err := mongoplus.TransactionCtx(ctx, db, func(txCtx context.Context) error {
    if err := l.purchaseOrderRepository.WithTx(txCtx).Create(ctx, orderEntity); err != nil {
        return err
    }
    return l.purchaseOrderDetailRepository.WithTx(txCtx).CreateBatch(ctx, detailEntities)
})
```

- WithTx 返回保留公开接口与扩展的副本，不改原仓储，也不开事务。必须使用返回值，如示例链式调用完整字段；不能丢弃返回值后对原仓储传普通 ctx。
- 绑定后普通方法继续传请求 ctx，查询、分页、聚合和写入共享 Session。实际版本允许直接向普通方法传回调 txCtx 时也可使用；不丢取消、租户、权限或操作人值。
- 校验通过绑定 Repository 取数据，在 Logic 判断再写入；不因分层移出事务。并发约束按 MongoDB 原子操作、唯一索引及实际隔离语义实现，不伪造 SQL 行锁。
- 每步错误返回回调，不只记日志后返回 nil。回调可能重试，每轮重新绑定；结束后不保存绑定仓储/txCtx 继续用，同 Session 内不并行操作。
- 所有参与仓储共用同 Client，事务要求副本集或分片集群。没有 SQL savepoint 或跨 MongoDB/MySQL 原子事务；不通过禁用校验、换后台 Context 或静默走事务外连接让调用通过。
- 基线无效绑定 ErrUnmanagedTransaction，Client/Session/轮次不符 ErrTransactionMismatch，作用域结束 ErrTxDone；按真实版本核对，不吞错误后调用普通方法。
- TransactionAsCtx(ctx,db,use,fn,opts...) 的 use 接收 context.Context，按需绑定工作单元，不照抄 gorm-gen 适配。短信、支付等不可重复外部动作不放可能重试的回调，沿项目已有提交后方案。

## Context 与框架

Gin 优先 c.Request.Context()，go-zero 用请求 ctx。已有项目传 *gin.Context 时检查 RegisterCtxResolver 及现有解析器，须保留标准 Context，尤其 txCtx；不在请求中重复注册。

租户、数据权限、逻辑删除与敏感字段优先复用插件和生成执行器。Native 原生调用不自动保证事务/缓存/插件行为；确需使用时只在 Repository 内封装数据操作，显式传真实 txCtx 并处理必要失效，不把原生 Collection 暴露给业务层绕过边界。

## 缓存按需检查

- 普通开发不主动加缓存。涉及缓存时核对真实查询键、租户/权限维度与数据源隔离，不搬 gorm-plus 的缓存限制或保证。
- 基线事务读跳过缓存与 SingleFlight，写入登记集合、提交后统一失效；回滚不发布，提交不确定时保守失效。验证真实提交及并发读写，不只看方法名。
- 多实例共享缓存复用已有 backend/Namespace 约定，同数据源一致、不同环境/数据源隔离；仅清本机内存不等于跨实例一致。
- WithTx 复制实现结构体，不可复制的锁等应通过指针持有；不按请求改全局默认库或并发改原仓储字段。

按修改验证成功提交、后续失败回滚、事务内读取及 Context；需要缓存/并发保证时再验证相应场景。无隔离测试库时说明未做真实事务验证。
