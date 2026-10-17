# mongo-plus-skill

面向使用 [`github.com/kuangshp/mongo-plus`](https://github.com/kuangshp/mongo-plus) 的 Go 项目，为 Codex 提供完整的开发、代码生成和排查规范，支持 Gin、go-zero 和普通 Go 服务。

这是完整独立的技能包，使用者不需要安装 `gorm-plus-skill` 或其他开发技能。

这是供 AI 编程助手使用的开发规则，不会替代业务项目安装 Go 依赖、配置数据库或执行迁移。

## 当前作用

- 从 `schema/*.json` 和项目配置生成 Entity、DAO、Repository，以及按需生成 DTO/VO。
- 区分离线代码生成与集合、校验器、索引同步，说明字段变化、迁移与生成覆盖规则。
- 规范 Repository 的模板复用、完整命名和业务对象构造函数初始化。
- 处理 ObjectID、BSON、Decimal128、嵌套对象、更新结果、范围查询、分页和聚合。
- 规范 MongoDB Session 事务、`WithTx(txCtx)`、Context、租户、数据权限、缓存与 DAL。
- 提供 Gin/go-zero 的连接生命周期、校验、HTTP 接入和验证示例。

默认开发流程：

```text
Schema JSON → 根目录 generator.go → 按配置同步或离线生成 → 初始化连接和必要插件 → 在业务对象构造函数初始化 Repository → 实现业务校验与接口 → 验证
```

Schema JSON 是生成输入；Model/Entity 是输出。未配置 MongoDB 时可以只生成代码，但必须明确数据库尚未同步。

## 核心约束

- **生成 Model/Entity 禁止手写、手改或手动新建文件**，字段、类型和标签通过生成输入、配置或模板修正后重新生成。
- **Repository 禁止手动新建文件**；模板已有能力优先复用，确实缺失的纯数据访问方法才能补充到对应已有非生成文件。
- **业务逻辑不放 Repository**；业务校验、权限/状态判断、计算、业务错误和流程编排由 Logic/Service 或项目已有业务层负责。
- **名称保持完整**，例如 `purchaseOrderRepository repository.PurchaseOrderRepository`，在业务对象构造函数中初始化，禁止改成 `repo` 等缩写。
- 单数据库且真实构造函数支持默认连接时可省略连接参数；多数据库参数由用户明确填写，不猜选库。
- 以当前项目实际依赖、生成结果和用户指令为准，不擅自升级库、改生产库、推送或部署。

完整约束以 [SKILL.md](SKILL.md) 为准，README 用于介绍与安装。

## 安装范围

以下方法适用于支持本地 Agent Skills 的 Codex。当前官方目录如下，参考 [OpenAI 官方技能文档](https://learn.chatgpt.com/docs/build-skills#where-codex-loads-local-skills)：

| 方式 | 安装位置 | 适用范围 |
| --- | --- | --- |
| 全局安装（当前用户） | `~/.agents/skills/mongo-plus-skill/` | 当前用户使用的各个项目 |
| 项目安装 | `<项目根目录>/.agents/skills/mongo-plus-skill/` | 当前仓库，可随项目提交给团队 |

全局和项目安装按需要选一种。同名技能不会自动合并，不依赖“项目版本必然覆盖全局版本”的假设。已有环境使用其他技能目录（例如此前配置的 `~/.codex/skills/`）时，以该客户端实际发现的目录为准，避免再复制一份同名技能。

### 全局安装

需要 Git；以下命令适用于 macOS、Linux 或 WSL 的终端，目标目录应尚不存在：

```bash
mkdir -p "$HOME/.agents/skills"
git clone https://github.com/kuangshp/mongo-plus-skill.git "$HOME/.agents/skills/mongo-plus-skill"
```

确认入口文件位于正确层级：

```bash
test -f "$HOME/.agents/skills/mongo-plus-skill/SKILL.md"
```

该命令成功时无输出。不要只下载 SKILL.md，还要保留 `references/` 和 `agents/`。

### 项目安装

先进入**要使用技能的业务项目根目录**，再执行下面命令。它会复制仓库已提交文件作为项目内快照，不会把技能仓库的 `.git` 目录放进业务项目；目标已存在时停止，避免覆盖本地定制。

```bash
(
  set -eu
  skill_dest=".agents/skills/mongo-plus-skill"
  if [ -e "$skill_dest" ]; then
    printf '%s\n' "$skill_dest 已存在，请先比较或备份后更新。" >&2
    exit 1
  fi
  skill_tmp_dir="$(mktemp -d)"
  trap 'rm -rf "$skill_tmp_dir"' EXIT
  git clone --depth 1 https://github.com/kuangshp/mongo-plus-skill.git "$skill_tmp_dir/source"
  git -C "$skill_tmp_dir/source" archive --format=tar --output="$skill_tmp_dir/skill.tar" HEAD
  mkdir -p "$skill_dest"
  tar -xf "$skill_tmp_dir/skill.tar" -C "$skill_dest"
  test -f "$skill_dest/SKILL.md"
)
```

项目内结构应为：

```text
your-project/
├── .agents/
│   └── skills/
│       └── mongo-plus-skill/
│           ├── README.md
│           ├── SKILL.md
│           ├── agents/openai.yaml
│           └── references/
└── ...
```

检查 `.agents/` 未被项目 `.gitignore` 忽略，再按团队流程将技能文件提交进业务仓库。团队成员拉取后，可在该项目中使用同一份规则；该安装不会改变其他项目。

若已在本地下载本技能，也可以直接将完整技能目录复制到上述位置，跳过 GitHub 下载；同样不要复制 `.git` 或只复制单个 SKILL.md。

## 使用方式

安装后在目标项目中打开 Codex。在 CLI/IDE 中可通过 `/skills` 查看，或在提示中显式写技能名；桌面端可在技能选择器中选取。也可直接输入：

```text
使用 $mongo-plus-skill，基于当前项目新增用户分页接口，复用生成的 Repository，保持现有路由、鉴权和响应格式。
```

Codex 也可根据项目依赖和任务描述自动匹配技能。新安装或修改通常会自动发现；若未出现，重新打开会话或重启 Codex。技能发现与调用方式见 [OpenAI 官方技能文档](https://learn.chatgpt.com/docs/build-skills)。

安装技能本身不安装数据库、不建立连接，也不自动执行 generator.go。使用时应说明业务需求、目标服务以及允许执行的生成或数据库操作。

## 更新与移除

全局安装若使用上述 Git clone 方式，确认安装目录没有需要保留的本地修改后更新：

```bash
git -C "$HOME/.agents/skills/mongo-plus-skill" pull --ff-only
```

项目安装是随业务仓库版本控制的快照。更新时取得上游新版本，与项目内修改比较后替换相应文件，再提交业务仓库；不要在快照目录运行 git pull，它可能作用到外层业务仓库。需要固定版本时，从指定 tag/commit 导出文件，不必始终跟随 main。

移除时删除所选范围中的 `mongo-plus-skill` 技能目录即可；项目安装的删除应随业务仓库提交。只操作技能目录，不影响数据库及 Go 依赖。

## 内容导航

- [完整技能规则](SKILL.md)
- [生成与接口流程](references/generator-workflow.md)
- [配置、Schema 与索引](references/schema-config.md)
- [运行时与 Gin/go-zero 接入](references/runtime-framework.md)
- [Repository 与查询](references/repository-query.md)
- [事务与上下文](references/transactions-context.md)
- [插件与 DAL](references/plugins-dal.md)
- [来源及验证记录](references/verification.md)
