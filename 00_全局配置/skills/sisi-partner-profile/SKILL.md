---
name: sisi-partner-profile
version: 1.0.0
description: "伙伴档案建立入口：根据用户提供的原始信息（可能是长文字稿），判断伙伴类型（重要客户 / 嘉宾 / 合作伙伴），在对应飞书知识库创建浓缩为一页的档案子页面，并在目录表格/客户名单中维护必要信息。"
metadata:
  requires:
    bins: ["lark-cli"]
  cliHelp: "lark-cli wiki --help;lark-cli docs --help;lark-cli base --help"
---

# sisi-partner-profile

## 何时使用

- 用户给出某人的原始信息（文字稿、聊天记录、访谈纪要、名片等），要求**建立档案页面**。
- 需要判断该人属于以下三类之一，并写到正确位置：
  - **重要客户**
  - **嘉宾**
  - **合作伙伴**
- 本 skill 不负责成交、权益、客户名单的日常增删改查；客户记录操作必须转 `sisi-client`。

## 前置读取（每次执行前必须做）

1. 读取 `00_全局配置/特殊客户档案管理.md`，确认三类档案的目标位置和命名规则。
2. 读取 `00_全局配置/飞书基本信息.md`，确认关键 ID：
   - **Base（库）Token**：`IWFEbuZcfalvQus6vkOcJXUjn2d`
   - **客户名单表 ID**：`tblvKLGIHObVQ3dV`

所有 `lark-cli` 命令必须显式使用 `--as user`。

## 父页面（知识库节点）速查

| 类型 | 父页面 URL | 父节点 token (`--parent-node-token`) | 父文档 obj_token |
|---|---|---|---|
| 重要客户 | `https://ghy685ffir.feishu.cn/wiki/MyoowL7HXibFFKkl1WdcOn2ende` | `MyoowL7HXibFFKkl1WdcOn2ende` | `C9jcdvkZGof2VjxJqQBceQrnnrb` |
| 嘉宾 | `https://ghy685ffir.feishu.cn/wiki/HuDmwW6Eni7RFfk8DaCcy1e0nbe?fromScene=spaceOverview` | `HuDmwW6Eni7RFfk8DaCcy1e0nbe` | `WY1BdQ2PHoqDWTxupIvc2Cnhnvh` |
| 合作伙伴 | `https://ghy685ffir.feishu.cn/wiki/Gpl5wybdpijRqEkLHdHcFmZznwh?fromScene=spaceOverview` | `Gpl5wybdpijRqEkLHdHcFmZznwh` | `Ur3TdvTUnovsYKx3NV8c6U0knWe` |

> 子页面 URL 形如 `https://ghy685ffir.feishu.cn/wiki/<node_token>`。创建后返回的 `node_token` 即可拼出 wiki 链接。

## 执行流程

### 1. 明确档案类型

先让用户明确或从上下文推断类型。如果用户没有说明，**必须询问**，禁止自行归类：

> “这是要建立为重要客户档案、嘉宾档案，还是合作伙伴档案？”

### 2. 收集并确认关键信息

根据类型，以下信息缺一不可；如果用户提供的是长文字稿，先由你（或必要时启动 SubAgent）从中提取并结构化：

| 类型 | 必填字段 | 用于档案的模块 |
|---|---|---|
| 重要客户 | 客户昵称、真实姓名、联系方式（手机号/微信号至少其一）、商业模式、当前问题/需求 | 基本信息、商业模式、当前问题 |
| 嘉宾 | 嘉宾昵称/姓名、背景与资历、可提供的供给/信息服务、联系方式（至少其一） | 嘉宾简介、可提供供给、对接方式 |
| 合作伙伴 | 合作伙伴昵称/姓名、可提供的服务/供给能力、合作方式、联系方式（至少其一） | 伙伴简介、供给能力、合作方式 |

如果信息不完整，**必须暂停并向用户索要**，禁止用占位符或猜测填充。

### 3. 查重（避免重复创建）

在目标父节点下列出已有子页面，确认是否已存在同名或同人的档案：

```bash
lark-cli wiki +node-list \
  --parent-node-token <父节点_TOKEN> \
  --as user \
  --format json
```

- 若已存在同名/同人页面：向用户确认是**更新已有档案**还是**创建新档案**。
- 若确认新建，继续下一步。

对于合作伙伴/重要客户，若涉及客户名单，还需调用 `sisi-client` 查重/确认客户记录（见步骤 5）。

### 4. 创建档案子页面

#### 4.1 创建节点

```bash
lark-cli wiki +node-create \
  --parent-node-token <父节点_TOKEN> \
  --title "<昵称> - <后缀>" \
  --as user \
  --format json
```

- 重要客户后缀：`档案`
- 嘉宾后缀：`嘉宾档案`
- 合作伙伴后缀：`合作伙伴档案`

记录返回的 `node_token`（wiki 链接用）和 `obj_token`（文档内容编辑用）。

#### 4.2 写入一页浓缩档案

使用 `docs +update --command overwrite` 将关键信息浓缩到一页内。内容建议结构：

- **重要客户**
  - 基本信息（昵称、姓名、联系方式、城市/行业）
  - 商业模式
  - 当前问题/需求
  - 关键结论/下一步
  - 原始信息摘要（如来自长稿，只保留要点）
- **嘉宾**
  - 嘉宾简介（身份、背景、核心领域）
  - 可提供的供给/信息服务
  - 覆盖维度 / 交付形式
  - 定价与分成（如有）
  - 对接方式
  - 案例与口碑 / 语料切片（如有）
- **合作伙伴**
  - 伙伴简介
  - 可提供的服务/供给能力
  - 覆盖领域 / 交付形式
  - 合作方式 / 对接流程
  - 定价与分成（如有）
  - 案例与口碑（如有）

示例 XML 内容（以嘉宾为例，按需替换）：

```xml
<title>杰哥 - 嘉宾档案</title>
<h1>嘉宾简介</h1>
<p>身份：小红书商业化操盘手、AI 营销讲师。</p>
<p>核心领域：小红书商业化、AI 营销、商业通识。</p>
<h1>可提供的供给</h1>
<ul>
  <li>小红书商业化 1v1 咨询</li>
  <li>AI 营销主题分享 / 工作坊</li>
</ul>
<h1>对接方式</h1>
<p>微信：xxx；提前 3 天预约。</p>
<h1>原始信息摘要</h1>
<p>来自访谈纪要：...</p>
```

写入命令：

```bash
lark-cli docs +update \
  --doc <OBJ_TOKEN> \
  --command overwrite \
  --content '<title>...</title><h1>...</h1>...' \
  --as user
```

### 5. 维护客户名单（仅重要客户、合作伙伴且同时为客户时）

#### 5.1 确认客户记录

**必须调用 `sisi-client`** 查询/确认客户：

- 若客户已存在：记录其 `record_id`。
- 若客户不存在：询问用户是否也要创建为客户；若需要，先通过 `sisi-client` 创建，拿到 `record_id` 后再继续。
- 若存在多条疑似记录：列出候选，请用户选择具体 `record_id`。

#### 5.2 更新客户名单

```bash
lark-cli base +record-upsert \
  --base-token IWFEbuZcfalvQus6vkOcJXUjn2d \
  --table-id tblvKLGIHObVQ3dV \
  --record-id <RECORD_ID> \
  --json '{"档案页":"https://ghy685ffir.feishu.cn/wiki/<NODE_TOKEN>"}' \
  --as user
```

- **合作伙伴且同时是客户**：在 `客户标签` 中追加 `合作伙伴`（注意保留已有标签）。
- 写入前用 `+field-list` 核对字段类型；`客户标签` 是多选，值用数组 `["VIP","合作伙伴"]`。

### 6. 更新目录表格（仅嘉宾、合作伙伴）

嘉宾/合作伙伴的父页面内分别有「嘉宾资源目录」「合作伙伴清单」表格，需要把新档案登记进去。

#### 6.1 获取父文档 obj_token

如果还没有 `obj_token`，用：

```bash
lark-cli wiki +node-get \
  --node-token "https://ghy685ffir.feishu.cn/wiki/<父节点_TOKEN>" \
  --as user \
  --format json
```

读取 `data.obj_token`。

#### 6.2 在表格末尾追加一行

获取父文档内容，找到目标表格的 `</tbody></table>`，在其前插入新行：

```bash
lark-cli docs +update \
  --doc <父文档_OBJ_TOKEN> \
  --command str_replace \
  --pattern '</tbody></table>' \
  --content '<tr><td vertical-align="top"><p>昵称</p></td><td vertical-align="top"><p>类别</p></td><td vertical-align="top"><p>核心领域</p></td><td vertical-align="top"><p>供给数量</p></td><td vertical-align="top"><p>状态</p></td><td vertical-align="top"><p><cite type="doc" doc-id="<档案_OBJ_TOKEN>" title="昵称"></cite></p></td></tr></tbody></table>' \
  --as user
```

- 表格列依次为：`嘉宾姓名`、`类别`、`核心领域`、`供给数量`、`状态`、`嘉宾页面`。
- 页面链接用 `<cite type="doc" doc-id="<档案_OBJ_TOKEN>"></cite>` 插入，可点击跳转。

> 如果父页面内存在多个 `</tbody></table>`，先用 `docs +fetch` 定位目标表格，再针对该表格的 block_id 做精确替换。

## 字段写入规范

写入 Base 前务必先用 `+field-list` 核对字段类型，再按以下规则构造 JSON：

| 字段类型 | 写入格式 | 示例 |
|---|---|---|
| text / phone / url | 字符串 | `"13800138000"` |
| select（单选） | 选项名字符串 | `"客户"` |
| select（多选） | 选项名数组 | `["VIP", "合作伙伴"]` |
| link | `[{"id": "<目标 record_id>"}]` | `[{"id": "recvkNdMOzezuP"}]` |

## 安全与错误处理

- **禁止未确认类型就创建**：若用户未明确类型，必须先问清楚。
- **信息不完整禁止写入**：缺少必填字段时，必须向用户索要，不可用占位符。
- **禁止重复创建**：创建前检查父节点下是否已有同名/同人子页面；涉及客户名单时先调用 `sisi-client` 查重。
- **禁止猜测 ID**：link 字段、select 选项必须来自真实命令返回。
- **多选标签不要覆盖**：更新 `客户标签` 时，先读取已有标签，再追加 `合作伙伴`。
- **scope / 权限错误**：如果 user 身份报授权不足，先转 `lark-shared` 处理认证，不要直接切换 `--as bot`。
- **常见错误码**：
  - `param baseToken is invalid`：检查是否错把 wiki token 当成 base token。
  - `1254045` 字段名不存在：重新 `+field-list`，使用真实字段名或字段 ID。
  - `ignored_fields` / `READONLY`：移除只读字段后重试。

## 输出要求

- 创建完成后，向用户报告：档案类型、子页面标题、wiki 链接、`node_token`、档案 `obj_token`。
- 若更新了客户名单，报告 `record_id` 及更新的字段。
- 若更新了目录表格，报告已在父页面表格中登记。
- 如果发现信息缺失、类型不明确或疑似重复，必须向用户确认后再继续，禁止静默跳过。
