---
name: sisi-content-asset
version: 1.0.0
description: "内容资产操作入口：新增、修改、查询内容资产，按活动和父子关系整理入库。核心数据源为飞书多维表格「团队核心数据库」中的「内容资产表」。"
metadata:
  requires:
    bins: ["lark-cli"]
  cliHelp: "lark-cli base --help"
---

# sisi-content-asset

## 何时使用

- 任何涉及**内容资产新增、修改、查询**的任务，都必须调用本 skill。
- 核心目标是准确维护「团队核心数据库」中的「内容资产表」，确保资产与活动正确关联、父子层级清晰。
- 本 skill 不处理客户信息；若涉及客户查询/新增，转 `sisi-client`。

## 前置读取（每次执行前必须做）

读取 `00_全局配置/飞书基本信息.md`，确认关键 ID：

- **Base（库）Token**：`IWFEbuZcfalvQus6vkOcJXUjn2d`
- **内容资产表 ID**：`tblFUT6iCNZEXfU2`
- **活动交付记录表 ID**：`tblZaAryoZeXyKEs`
- **团队成员表 ID**：`tbl7HlE2tDnNCbwC`

所有命令必须显式使用 `--as user`。

## 核心流程：批量管理资产入库

### 1. 确认活动交付记录

资产必须关联到已有的活动交付记录。**不允许在内容资产表内凭空创建活动信息**。

#### 1.1 按活动标题搜索

```bash
lark-cli base +record-search \
  --base-token IWFEbuZcfalvQus6vkOcJXUjn2d \
  --table-id tblZaAryoZeXyKEs \
  --keyword "<活动标题>" \
  --search-field 活动标题 \
  --limit 20 \
  --format json \
  --as user
```

- **命中 1 条**：记录其 `record_id`，作为后续 `关联活动` 的值。
- **命中 0 条**：活动不存在。询问用户是否新建该活动交付记录；若用户不确认，**停止操作**，告知用户先去创建活动交付记录。
- **命中多条**：列出候选活动的「活动标题 + 开始时间 + 活动类型 + record_id」，请用户选择，禁止擅自决定。

#### 1.2 新建活动交付记录（仅在用户明确同意时执行）

若用户确认新建，收集以下信息：

| 字段 | 是否必填 | 说明 |
|---|---|---|
| 活动标题 | **必填** | 如「26年6月16北京商业下午茶」 |
| 活动类型 | **必填** | 默认「商业下午茶」，可选：线上连麦 / 群内答疑 / 商业下午茶 / 播客闭门会 / 线下课 / 其他 |
| 状态 | **必填** | 默认「已完成」，可选：计划中 / 已完成 / 已取消 |
| 开始时间 | 可选 | `YYYY-MM-DD HH:mm` |
| 结束时间 | 可选 | `YYYY-MM-DD HH:mm` |
| 负责人 | 可选 | link → 团队成员，需先解析 record_id |
| 参与客户 | 可选 | link → 客户名单，需先解析 record_id |

执行：

```bash
lark-cli base +record-upsert \
  --base-token IWFEbuZcfalvQus6vkOcJXUjn2d \
  --table-id tblZaAryoZeXyKEs \
  --json '<活动交付记录 JSON>' \
  --as user
```

记录返回的 `record_id`。

### 2. 确认或创建汇总记录

查找该活动是否已有内容资产汇总记录：

```bash
lark-cli base +record-list \
  --base-token IWFEbuZcfalvQus6vkOcJXUjn2d \
  --table-id tblFUT6iCNZEXfU2 \
  --filter-json '{"logic":"and","conditions":[["关联活动","==",["<活动 record_id>"]],["资产类型","==","汇总记录"]]}' \
  --format json \
  --as user
```

- **命中 1 条**：记录其 `record_id`，作为子资产的父记录。
- **命中 0 条**：创建一条新的汇总记录。
- **命中多条**：向用户展示并确认使用哪一条；无法确认时停止。

#### 新建汇总记录

```bash
lark-cli base +record-upsert \
  --base-token IWFEbuZcfalvQus6vkOcJXUjn2d \
  --table-id tblFUT6iCNZEXfU2 \
  --json '<汇总记录 JSON>' \
  --as user
```

汇总记录 JSON 示例：

```json
{
  "资产名称": "26年6月16日北京商业下午茶",
  "资产类型": ["汇总记录"],
  "关联活动": [{"id": "<活动 record_id>"}],
  "录制日期": "2026-06-16"
}
```

记录返回的 `record_id`，作为子资产的父记录。

### 3. 批量新增/更新子资产

根据用户提供的资料清单，逐条处理。用户提供什么就管理什么，**不强制要求任何字段**。

#### 3.1 资产类型识别

用户提供的资料应归入以下类型之一：

| 用户描述关键词 | 资产类型 |
|---|---|
| 音频、录音、粗剪音频、粗拼音频 | 粗拼音频 |
| 视频、录像、原始视频 | 原始视频 |
| 文稿、逐字稿、原文稿、文字稿 | 文字稿 |
| 总结、摘要、纪要 | 总结 |
| 剪辑、片段、精剪 | 剪辑片段 |
| 海报、封面、宣传图 | 海报 |
| 不属于以上类别 | 其他 |

若用户描述不清晰，**必须询问确认**后再写入。

#### 3.2 子资产字段构造

每条子资产必须设置：

| 字段 | 说明 |
|---|---|
| 资产名称 | 用户提供的名称，如「音频」、「森林文稿」 |
| 资产类型 | 按上表识别后的类型 |
| 父记录 4 | `[{"id": "<汇总记录 record_id>"}]` |
| 关联活动 | `[{"id": "<活动 record_id>"}]`（可选但建议一致） |

可选字段（用户提供什么就写什么）：

| 字段 | 说明 |
|---|---|
| 云盘链接 | 外部存储链接 |
| 下午茶咨询文稿 | 附件字段，上传原文稿 |
| 下午茶咨询问题描述 | 附件字段，上传问题描述 |
| 录制日期 | `YYYY-MM-DD` |
| 可发布授权 | 默认 `false`；只有用户明确说「已获得授权」时才写 `true` |
| 客户特殊需求 | 如「所讲内容不能外传」 |
| 备注 | 补充说明 |

#### 3.3 子资产写入

```bash
lark-cli base +record-upsert \
  --base-token IWFEbuZcfalvQus6vkOcJXUjn2d \
  --table-id tblFUT6iCNZEXfU2 \
  --json '<子资产 JSON>' \
  --as user
```

子资产 JSON 示例（文字稿）：

```json
{
  "资产名称": "森林文稿",
  "资产类型": ["文字稿"],
  "父记录 4": [{"id": "<汇总记录 record_id>"}],
  "关联活动": [{"id": "<活动 record_id>"}],
  "可发布授权": false,
  "客户特殊需求": "所讲内容不能外传",
  "录制日期": "2026-06-16"
}
```

#### 3.4 附件上传

附件字段必须通过专用命令上传，**不能**作为普通字段写入。

```bash
lark-cli base +record-upload-attachment \
  --base-token IWFEbuZcfalvQus6vkOcJXUjn2d \
  --table-id tblFUT6iCNZEXfU2 \
  --record-id <子资产 record_id> \
  --field 下午茶咨询文稿 \
  --file "/path/to/file.docx" \
  --as user
```

- 用户未提供附件则不传。
- 用户提供了多个附件，可多次调用或一次上传多个文件（按 CLI 实际支持）。

### 4. 查询内容资产

#### 4.1 按活动查询全部资产

```bash
lark-cli base +record-list \
  --base-token IWFEbuZcfalvQus6vkOcJXUjn2d \
  --table-id tblFUT6iCNZEXfU2 \
  --filter-json '{"logic":"and","conditions":[["关联活动","==",["<活动 record_id>"]]]}' \
  --format json \
  --as user
```

#### 4.2 按资产类型查询

```bash
lark-cli base +record-list \
  --base-token IWFEbuZcfalvQus6vkOcJXUjn2d \
  --table-id tblFUT6iCNZEXfU2 \
  --filter-json '{"logic":"and","conditions":[["资产类型","==","文字稿"]]]}' \
  --format json \
  --as user
```

#### 4.3 按名称模糊查询

```bash
lark-cli base +record-search \
  --base-token IWFEbuZcfalvQus6vkOcJXUjn2d \
  --table-id tblFUT6iCNZEXfU2 \
  --keyword "<资产名称>" \
  --search-field 资产名称 \
  --limit 20 \
  --format json \
  --as user
```

### 5. 更新内容资产

1. 先按 `record_id` 或活动 + 名称定位到目标记录。
2. 若命中多条或不确定，必须请用户选择。
3. 只更新用户明确要求修改的字段，`--json` 中不要包含无关字段。
4. 附件更新走 `+record-upload-attachment`；普通字段更新走 `+record-upsert`。

## 字段写入规范

写入前务必先用 `lark-cli base +field-list` 核对字段类型，再按以下规则构造 `CellValue`：

| 字段类型 | 写入格式 | 示例 |
|---|---|---|
| text / url | 字符串 | `"https://pan.baidu.com/..."` |
| select（单选） | 选项名字符串 | `"文字稿"` |
| select（多选） | 选项名数组 | `["粗拼音频", "原始视频"]` |
| datetime | `YYYY-MM-DD` 或 `YYYY-MM-DD HH:mm:ss` | `"2026-06-16"` |
| link | `[{"id": "<目标 record_id>"}]` | `[{"id": "recvme653iGTsV"}]` |
| checkbox | boolean | `true` / `false` |
| attachment | 走 `+record-upload-attachment` | — |

### 只读字段（禁止写入）

- `创建日期`（created_at，系统自动维护）

如果返回中出现 `ignored_fields` 或 `READONLY`，立即移除对应字段再重试。

## 内容资产表字段速查

| 字段名 | 字段 ID | 类型 | 说明 |
|---|---|---|---|
| 资产名称 | `fldzqUskOU` | text | **必填** |
| 资产类型 | `fldW8j6mdt` | multi-select | 粗拼音频 / 原始视频 / 文字稿 / 总结 / 剪辑片段 / 海报 / 其他 / 汇总记录 |
| 关联活动 | `fldDIZdu6b` | link → 活动交付记录 | **必填** |
| 父记录 4 | `fldZPIkjkQ` | link → 内容资产表 | 子资产指向汇总记录 |
| 云盘链接 | `fldr8AB2FJ` | text/url | 外部存储链接 |
| 下午茶咨询文稿 | `fldb6T7Mqo` | attachment | 原文稿附件 |
| 下午茶咨询问题描述 | `fldGpYrGtQ` | attachment | 问题描述附件 |
| 录制日期 | `fldfT6JGR0` | datetime | `YYYY-MM-DD` |
| 可发布授权 | `fldwrsEK4E` | checkbox | **默认 `false`** |
| 客户特殊需求 | `fldrYVTvzv` | text | 保密要求等 |
| 备注 | `fldO1rlEun` | text | 补充说明 |
| 创建日期 | `flddScCr9O` | created_at | 通常只读 |

## 关联表解析

| 本表字段 | 目标表 | 目标表 ID | 解析字段 |
|---|---|---|---|
| 关联活动 | 活动交付记录 | `tblZaAryoZeXyKEs` | 活动标题 / 开始时间 / 活动类型 |
| 父记录 4 | 内容资产表 | `tblFUT6iCNZEXfU2` | 资产名称 + 资产类型 = 汇总记录 |
| 负责人（活动交付记录） | 团队成员 | `tbl7HlE2tDnNCbwC` | 昵称 / 真实姓名 |

解析命令示例：

```bash
lark-cli base +record-search \
  --base-token IWFEbuZcfalvQus6vkOcJXUjn2d \
  --table-id tblZaAryoZeXyKEs \
  --keyword "<活动标题>" \
  --search-field 活动标题 \
  --limit 20 \
  --format json \
  --as user
```

## 安全与错误处理

- **必须先确认活动**：内容资产写入前，必须先确认活动交付记录存在；不存在时必须询问用户是否新建，用户不确认则停止。
- **禁止重复创建汇总记录**：同一活动只能有一条「汇总记录」；新增前必须先查询，命中多条时向用户确认。
- **父子关系必须正确**：所有子资产必须通过「父记录 4」关联到汇总记录，不允许孤立记录。
- **可发布授权默认 false**：只有用户明确授权时才写 `true`。
- **不强制附件**：用户提供什么附件就上传什么，不主动要求补全「原文稿 + 问题描述」。
- **禁止猜测 ID**：link 字段、select 选项必须来自真实命令返回。
- **scope / 权限错误**：如果 user 身份报授权不足，先转 `lark-shared` 处理认证，不要直接切换 `--as bot`。
- **常见错误码**：
  - `param baseToken is invalid`：检查是否错把 wiki token 或 space_id 当成 base token。
  - `1254045` 字段名不存在：重新 `+field-list`，使用真实字段名或字段 ID。
  - `ignored_fields` / `READONLY`：移除只读字段后重试。

## 输出要求

- 查询结果展示时，核心信息包含：`资产名称`、`资产类型`、`父记录 4` 指向的汇总记录、可发布授权；可附带 `record_id` 供后续操作引用。
- 批量入库完成后，向用户确认：活动名称、汇总记录 `record_id`、本次新增/更新的资产清单（名称 + 类型 + record_id）。
- 任何不确定或疑似重复/冲突时，必须停下来向用户确认，禁止静默决定。
