# 配置、Schema 与索引

## 最小可执行的配置

项目已依赖该库时不重新安装或升级；新项目按用户选定或验证过的版本加入 `github.com/kuangshp/mongo-plus`。依赖版本、Go 工具链及 CLI 版本核对实际 go.mod，不无条件使用 latest。生成本身只需要公共 Go 包，无需安装 CLI 或其他技能。

此示例假定 go.mod 的 module 是 `example.com/service`，实际项目替换为自己的值。按本包生成流程准备 generator.go，保存以下内容为模块根目录 mongo-plus.yaml：

```yaml
package: example.com/service
schemaDir: schema
model_pkg_path: query/model
out_path: query/dao
repo_path: query/repository
dto_path: internal/dto
vo_path: internal/vo
pluralize: false
id:
  type: objectId
softDelete: true
timestamps: true
```

上面不带数据源的配置仅用于用户明确要求的离线生成/模板验证，不满足完整业务接口开发的前置条件。配合本包 `schema/user.json`，在模块根目录运行 `go run generator.go mongo-plus.yaml`。这是完整离线生成配置，**未同步数据库**；`dto_path`/`vo_path` 留空则不生成对应契约文件。复用现有目录，不因示例另建第二套。

业务接口开发必须先实际创建/同步集合及结构，因此应在同一生成配置中增加以下数据源段，或复用项目已有的显式同步入口并核对真实结果；环境变量由部署/开发环境提供，不在代码中记录真实凭据：

```yaml
mongodb:
  uri: "${MONGO_URI}"
  database: "${MONGO_DATABASE}"
  maxPoolSize: 100
```

仍执行同一生成入口，公共 Generate 会先同步后生成。不要将空变量、不可达连接或错误数据库当成离线模式。库 Config.Timeout 是 Go time.Duration，不能随意在 YAML 填一个不支持的 timeout 字段；运行时若采用 TimeoutSeconds，须由项目配置映射并校验单位。

## Schema 输入规则

| 属性 | 用途 |
| --- | --- |
| entity / collection / comment | 可覆盖实体类型名、真实集合名与注释；未给时按实际命名规则推导 |
| id | 可按集合覆盖全局主键类型与生成方式 |
| timestamps / softDelete / tenant | 覆盖相应全局默认值；tenant 按配置声明 name/type |
| fields | 字段数组，name/type 必需，可设 goName/comment/required/nullable/unique/rules |
| fields[].fields | 嵌套 object 或 []object 的子字段 |
| indexes | 普通、复合、唯一、TTL 等索引声明 |
| options | 当前版本支持的集合选项和校验模式 |

`required` 是数据库字段必须存在，与 HTTP 非空校验不是同一个概念。类型、规则、重复字段或自动字段冲突会报错，不通过手改实体消除。未知 Schema 属性不能猜填；附加长度/范围等约束放 rules。

## 主键与字段类型

| Schema 类型 | Entity Go 类型/约束 |
| --- | --- |
| objectId | bson.ObjectID；默认 auto，可显式 manual |
| int64 主键 | 默认 increment，自增号码允许间断，不保证连续业务编号；manual 由调用者提供 |
| string 主键 | 默认 manual，不隐式生成字符串 ID |
| string / int32 / int64 / bool / float64 | 对应 Go 类型；float64 不用于需要精确十进制的金额 |
| datetime / decimal128 | time.Time / bson.Decimal128；HTTP 中用字符串校验解析 |
| bytes / json | []byte / map[string]any |
| object / []object | 生成嵌套结构体及切片，字段在 fields 中声明 |
| []加受支持的基础类型 | 对应切片，按当前版本检查支持范围 |

nullable 会影响生成 Go 类型，按真实输出核对指针。ID 默认 objectId，不假设与请求中的 string 或其他集合 int64 引用自动兼容。自动创建/更新时间和 deleted_at 等字段由生成配置维护，不手动重复定义。

## 嵌套与索引示例

下面为另一种 user.json，替换最小定义使用，不同时维护同集合的第二份 JSON：

```json
{
  "entity": "UserEntity",
  "collection": "users",
  "fields": [
    {"name": "username", "type": "string", "required": true, "rules": {"minLength": 2}},
    {"name": "status", "type": "int64", "rules": {"enum": [0, 1, 2]}},
    {"name": "balance", "type": "decimal128"},
    {"name": "tags", "type": "[]string"},
    {"name": "profile", "type": "object", "nullable": true, "fields": [
      {"name": "city", "type": "string"}
    ]},
    {"name": "lines", "type": "[]object", "fields": [
      {"name": "sku_id", "type": "objectId"},
      {"name": "quantity", "type": "int64"}
    ]}
  ],
  "indexes": [
    {"name": "uk_username_active", "fields": [{"name": "username", "order": 1}], "unique": true, "activeOnly": true},
    {"name": "idx_status_created", "fields": [{"name": "status", "order": 1}, {"name": "created_at", "order": -1}]}
  ]
}
```

- 复合键按 fields 数组顺序，不用无序 map 代替。租户唯一性常需租户字段参与复合唯一索引，不能由全局用户名唯一索引冒充。
- activeOnly 与软删除配置关联；启用前核对当前版本与历史重复数据。required/唯一索引不能替代业务校验，业务拒绝仍在业务层。
- TTL 使用 expireAfterSeconds 和当前支持的索引类型/日期字段，过期删除非精确计时，不承担必须准点执行的业务流程。
- 索引命名和等价性核对真实计划；冲突不默认删除重建。新增唯一索引前检查存量重复，严格校验规则变更前检查旧数据。

## 显式迁移与排错

Schema 同步不会删除旧字段或回填历史文档；字段重命名、类型转换、索引替换、特殊集合等使用项目受版本控制的显式迁移。库内 `Migration` 提供 Version/Description/Checksum/Up/Down，按 Version 排序；Checksum 应来自确定的迁移源码，不按当前时间生成。迁移失败先检查已完成操作与历史记录，不自动开启 ResumeFailed。

连接失败检查配置来源及 URI 的认证/副本集参数；生成路径错误检查 Go 工作目录与 YAML 相对目录；产物未变检查 SchemaDir、文件名和保留策略。日志不能打印完整 URI/凭据，不能临时改 Model 或库内代码让生成通过。
