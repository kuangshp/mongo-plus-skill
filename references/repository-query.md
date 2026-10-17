# Repository、查询与请求类型

## 复用与职责边界

先检查当前默认接口与嵌入：`UserRepository` 嵌入 `iDefaultUserRepository`，`customerUserRepository` 嵌入 `*defaultUserRepository` 时，默认 CRUD 已可用。名称及签名以实际生成结果为准。

标准操作能用方法、Wrapper 或选项表达就直接调用；不要因 HTTP 名称不同再定义 CreateUser/DeleteUser/RenameUser，不仅为 Req 包装一行 CRUD。映射复用现有 Mapper/Logic，不把完整 HTTP 请求契约带入 Repository。

确需扩展时说明已有能力与缺口，只在对应集合已有非生成文件新增纯数据方法并更新公开接口。缺文件先修复生成流程，不手动新建 helper/query/join/extra 或第二套实现。联表、聚合属数据访问；业务判断、计算、错误、跨仓储流程属业务层，禁止通过私有函数、模板或 CRUD 覆盖隐藏规则。

## 常用方法与返回值

| 需求 | 基线能力及注意事项 |
| --- | --- |
| 创建/批量创建 | Create/CreateBatch 返回 error，成功回填自动字段，不直接返回新 ID |
| 按 ID、条件取单条 | FindById、FindOne、FindOneWrapper；用 errors.Is(err, mongoplus.ErrNotFound) 区分未找到 |
| 列表/分页 | FindListByWrapper、FindPageByWrapper 等；分页返回列表、total、err |
| 是否存在/数量 | ExistsByWrapper、CountByWrapper 返回事实，业务层决定能否继续 |
| 更新 | UpdateById、UpdateMapById、UpdateByWrapper 返回 *mongo.UpdateResult 和 error，区分 MatchedCount/ModifiedCount |
| 删除/恢复 | DeleteById、DeleteByIdList、RestoreById 等返回 int64、error；按契约选逻辑/物理删除 |
| 原子查改/查删 | 先查 FindOneAndUpdate/FindOneAndDelete 等，不自行查询后更新替代原子操作 |

调用以本表及实际生成签名为准，不借用其他库的方法契约。先处理 error 再使用结果；匹配但值未变化不自动等于未找到，删除零条的业务含义也在业务层处理。批量创建不保证整批回滚，需要时用事务；失败不可盲重试造成重复写入。

## BSON 与输入转换

- 核对真实 ID 类型 ObjectID/int64/string。HTTP ObjectID 字符串在业务层用 `bson.ObjectIDFromHex` 校验解析，引用字段 BSON 类型需匹配，不能依赖隐式转换。
- 基线驱动为 `go.mongodb.org/mongo-driver/v2/bson`，不混旧驱动 primitive；按实际依赖使用。Decimal128 金额按契约用 `bson.ParseDecimal128` 解析，不经 float64 兜转。
- 请求时间字段全用 string，在 binding/validate 标签添加 dateTime 并注册、接入请求校验。先复用库/项目现有 validation 工具，不假设标签自动执行；业务层按项目时区解析 time.Time。
- 生成 DTO 可手改，实体不可。必填数字、bool、数组可能用指针区分缺失与合法 0/false/空集合，不以零值判定是否传参；不假设 nullable、缺失字段、BSON null 相同。嵌套路径核对真实 DAO。
- 更新先用字段 Value/Set 和已有 UpdateOption，保留零值；不把完整 DTO 转成覆盖文档误清未提交字段。Modify DTO 是完整提交模板，不自动具有 PATCH 语义。

## 条件与范围

- 基线回调 `func(w *mongoplus.Wrapper)`，字段完整写 `dao.UserEntity.Status`，不定义实体别名；每次查询创建自己的 Wrapper，不跨 goroutine 复用构造中对象。
- WhereIf 只控制附加条件，实参仍先求值，不在 false 条件下也解引用 nil 请求指针。
- Like 包含、LLike 后缀、RLike 前缀匹配，空字符串跳过；实现会转义正则特殊字符，SQL 的 %/_ 是普通字符。
- BetweenIfNotZero 任一端点为零值就跳过整个范围，双端非零时包含两端。成对非零范围优先用它；契约允许单端或合法 0 边界时明确用字段 Gte/Lte，不静默漏筛选；时间先校验顺序并解析。
- AND/OR 混合显式分组，不假设 OrWhereIf(true,a,b) 表示 OR (a AND b)。
- 原生 BSON 只由可信 Repository 代码构造，先复用 RawWhere/WhereRaw，不将前端任意操作符对象透传；不为绕过无条件写防护随意加 AllowAll。

## 分页与聚合

- 普通分页用 FindPage/FindPageByWrapper(ctx,pageNumber,pageSize,...)，基线已内部调用 DbPage，业务层不再次算 offset 当页码传入。
- 必须直接算偏移时才用 `offset, limit := mongoplus.DbPage(req.PageNumber, req.PageSize)`；返回 int64，默认第一页、10 条、最多 500 条。不手算，输入校验仍按契约。
- 联表/投影优先 Repository.WithContext(ctx).LeftJoin/Lookup 与 ScanByPage；纯查询放对应已有 Repository 文件，不自建 Count/Skip/Limit，不搬 GORM 的 Select/As 签名。
- `query.ScanByPage(&rows, page, size)` 返回 total、err；`mongoplus.ScanByPage[VO](ctx, query, page, size)` 返回 rows、total、err。VO 按真实输出别名和 BSON 标签定义，不修改实体凑投影。
- 核对关联字段、租户与逻辑删除。LeftJoin 展开一对多可改变 total；列表与计数默认两次查询，不承诺天然一致快照。
- 大集合深分页按需用 FindCursorPage，批量导出可用 Walk；不假设游标支持任意排序或自带 total，不无条件改接口契约。

## 可直接对照的调用形状

以下为已有业务方法内的模板方法调用片段，字段已在构造函数初始化；默认示例使用 ObjectID，业务项目应使用自己的主键类型。创建、映射和业务异常留在业务层，无需新增 Repository 方法。

```go
entity := &model.UserEntity{Username: req.Username, Status: req.Status}
if err := l.userRepository.Create(l.ctx, entity); err != nil {
    return err
}
// 默认 ObjectID 主键在成功后回填 entity.ID；不是 Create 的第二个返回值。
id := entity.ID
result, err := l.userRepository.UpdateById(l.ctx, id,
    mongoplus.Update().WithColumns(dao.UserEntity.Status.Set(0)).Build(),
)
if err != nil {
    return err
}
_ = result.MatchedCount // 是否允许未命中由业务契约决定。
affected, err := l.userRepository.DeleteById(l.ctx, id)
if err != nil {
    return err
}
_ = affected
```

分页条件作为参数交给模板方法；这是复用模板的查询参数构造，不是业务层直接调用 DAO/驱动执行查询：

```go
rows, total, err := l.userRepository.FindPageByWrapper(l.ctx,
    req.PageNumber, req.PageSize,
    func(w *mongoplus.Wrapper) {
        w.Like(dao.UserEntity.Username, req.Keyword).
            BetweenIfNotZero(dao.UserEntity.CreatedAt, startTime, endTime).
            OrderDefault(dao.UserEntity.ID.Desc())
    },
)
```

req、时间解析值及错误处理使用当前接口定义，不能将此片段的字段假定为所有请求都存在。模板无法表达的复杂聚合仍放在对应已有 Repository 文件。

需要校验与写入原子性时，业务层按[事务与上下文](transactions-context.md)组合绑定 Repository，不把规则塞入数据层。DTO 类型及校验接入见[运行时与框架](runtime-framework.md)。
