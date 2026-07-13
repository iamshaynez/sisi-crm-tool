---
name: sisi-client
version: 1.0.0
description: "客户信息操作入口：新增、修改、查询客户名单。所有客户信息操作必须走本 skill，核心数据源为飞书多维表格「团队核心数据库」中的「客户名单」表。"
metadata:
  requires:
    bins: ["lark-cli"]
  cliHelp: "lark-cli base --help"
---

# sisi-client

## 何时使用

- 任何涉及**新增、修改、查询客户信息**的任务，都必须调用本 skill。
- 核心目标是准确维护「团队核心数据库」中的「客户名单」表。
- 准确性优先：新增/更新前必须先查重、查相似，确认目标后再写入；禁止猜测、禁止混淆客户、禁止重复添加。

## 前置读取（每次执行前必须做）

读取 `00_全局配置/飞书基本信息.md`，确认关键 ID：

- **Base（库）Token**：`IWFEbuZcfalvQus6vkOcJXUjn2d`
- **客户名单表 ID**：`tblvKLGIHObVQ3dV`
- **库 URL**：`https://ghy685ffir.feishu.cn/base/IWFEbuZcfalvQus6vkOcJXUjn2d`

所有命令必须显式使用 `--as user`。

## 客户唯一标识规则

客户表没有单一主键。以下字段都可作为识别依据：

`客户昵称`、`真实姓名`、`手机号`、`微信号`、`微博账号`、`抖音账号`、`小红书账号`、`小宇宙账号`、`视频号账号`、`其他社交账号`

查询时优先使用用户给出的最具体标识（如手机号、微信号）。

## 查询客户

### 推荐命令

```bash
lark-cli base +record-search \
  --base-token IWFEbuZcfalvQus6vkOcJXUjn2d \
  --table-id tblvKLGIHObVQ3dV \
  --keyword "<客户标识>" \
  --search-field 客户昵称 \
  --search-field 真实姓名 \
  --search-field 手机号 \
  --search-field 微信号 \
  --search-field 微博账号 \
  --search-field 抖音账号 \
  --search-field 小红书账号 \
  --search-field 小宇宙账号 \
  --search-field 视频号账号 \
  --search-field 其他社交账号 \
  --limit 50 \
  --format json \
  --as user
```

### 精确匹配

如果知道具体字段值，优先用 `+record-list` 做精确筛选：

```bash
lark-cli base +record-list \
  --base-token IWFEbuZcfalvQus6vkOcJXUjn2d \
  --table-id tblvKLGIHObVQ3dV \
  --filter-json '{"logic":"and","conditions":[["微信号","==","xxx"]]}' \
  --format json \
  --as user
```

### 查询后处理

- 命中 **0 条**：当前关键词未找到匹配；但新增/更新前仍需按「查重流程」用更宽泛方式再查一次。
- 命中 **1 条**：记录其 `record_id`，但仍需结合查重流程确认是否为唯一匹配。
- 命中 **多条**：列出匹配记录的关键字段（客户昵称、真实姓名、手机号、微信号，并附带 record_id 供选择），请用户选择，不要擅自决定。
- **命中 0 条但存在相似记录**（如昵称前缀/后缀相同、手机号差一位、微信号相似、社交账号部分一致）：视为疑似重复，必须展示并确认。

## 查重流程（新增/修改前必须执行）

无论是用户说「新增客户」还是「更新客户信息」，在写入前都必须执行本流程，避免重复或误改。

1. **收集用户给出的所有识别信息**：`客户昵称`、`真实姓名`、`手机号`、`微信号`、`微博账号`、`抖音账号`、`小红书账号`、`小宇宙账号`、`视频号账号`、`其他社交账号`。
2. **对每个识别信息分别执行 `+record-search`**，关键词使用原始值，也尝试去掉空格、特殊符号后的变体。
3. **对昵称/账号类字段做模糊/子串匹配**：例如用户给「Mandy」，也要搜包含「Mandy」的昵称/账号；给「adventure44」，也要搜可能的变体。
4. **汇总所有命中记录**，去重后按相似度排序。
5. **判断**（必须向用户确认后再继续）：
   - 有任意字段**完全匹配**（如手机号、微信号相同）→ 视为同一客户，**不能新增，只能更新**；向用户说明并确认。
   - 有**高度相似但不完全匹配** → 列出疑似记录，请用户确认是「同一人/更新」还是「不同人/新增」。
   - **完全无相似** → 方可进入新增流程。

## 新增客户

1. **即使用户明确说「新增客户」，也必须先执行「查重流程」**。未确认无重复前不得写入。
2. 收集客户信息，至少应包含一个可识别字段（推荐 `客户昵称`）。
3. 构造 `--json` 时严格按字段类型写入（见下方「字段写入规范」）。
4. 执行：

```bash
lark-cli base +record-upsert \
  --base-token IWFEbuZcfalvQus6vkOcJXUjn2d \
  --table-id tblvKLGIHObVQ3dV \
  --json '<客户字段 JSON>' \
  --as user
```

5. 记录返回的 `record_id`。

## 修改客户

1. **即使用户明确说「更新某客户」，也必须先执行「查重流程」**，找到疑似匹配记录。
2. 根据查重结果处理：
   - **只有 1 条高度匹配** → 记录其 `record_id`，用于更新。
   - **0 条匹配** → 向用户确认是否信息有误；若用户确认是已存在客户但暂时找不到，可请用户提供更多标识，或按新增处理（需用户明确同意）。
   - **多条匹配或不确定** → 必须列出候选记录的关键字段，请用户选择具体 `record_id` 后才能更新。
3. 只更新用户明确要求修改的字段，`--json` 中不要包含无关字段。
4. 执行：

```bash
lark-cli base +record-upsert \
  --base-token IWFEbuZcfalvQus6vkOcJXUjn2d \
  --table-id tblvKLGIHObVQ3dV \
  --record-id <record_id> \
  --json '<待更新字段 JSON>' \
  --as user
```

## 字段写入规范

写入前务必先用 `lark-cli base +field-list --base-token IWFEbuZcfalvQus6vkOcJXUjn2d --table-id tblvKLGIHObVQ3dV --as user --format json` 核对字段类型，再按以下规则构造 `CellValue`：

| 字段类型 | 写入格式 | 示例 |
|---|---|---|
| text / phone / url | 字符串 | `"13800138000"` |
| select（单选） | 选项名字符串 | `"客户"` |
| select（多选） | 选项名数组 | `["VIP", "业务阶段 1-10"]` |
| datetime | `YYYY-MM-DD HH:mm:ss` | `"2026-07-07 15:30:00"` |
| link | `[{"id": "<目标 record_id>"}]` | `[{"id": "recvkNdMOzezuP"}]` |
| checkbox | boolean | `true` / `false` |

### 只读字段（禁止写入）

- `归属人_飞书用户`（lookup 字段，id: `fldaT4xyDG`）
- `创建日期`（id: `fldygfdu3o`）通常由系统自动维护，除非用户明确要求覆盖，否则不写。

如果返回中出现 `ignored_fields` 或 `READONLY`，立即移除对应字段再重试。

## 关联表解析

当用户给出 link 字段的值（如归属人、来源渠道）时，不要直接写名称，必须先去对应表中解析出 `record_id`：

| 本表字段 | 目标表 | 目标表 ID |
|---|---|---|
| 归属人 | 团队成员 | `tbl7HlE2tDnNCbwC` |
| 升单人 | 团队成员 | `tbl7HlE2tDnNCbwC` |
| 来源渠道 | 渠道资产 | `tblx3PGzNONP3Ugk` |
| 所在社群 | 渠道资产 | `tblx3PGzNONP3Ugk` |
| 交付记录 | 用户权益明细表 | `tbl5PQgyV5vfKia0` |
| 客户跟进记录 | 客户跟进记录 | `tblI8KxzaJW7lXTL` |
| 线索和成交记录 | 成交表 | `tblU9xWQEC9vt5yb` |
| 父记录 | 客户名单 | `tblvKLGIHObVQ3dV` |

解析命令示例：

```bash
lark-cli base +record-search \
  --base-token IWFEbuZcfalvQus6vkOcJXUjn2d \
  --table-id tbl7HlE2tDnNCbwC \
  --keyword "<成员名>" \
  --search-field 姓名 \
  --limit 20 \
  --format json \
  --as user
```

若目标表字段名不是「姓名」，先用 `+field-list` 确认可搜索字段。

## 客户名单字段速查

| 字段名 | 字段 ID | 类型 | 说明 |
|---|---|---|---|
| 客户昵称 | `fld19WQTw4` | text | 主要展示名称 |
| 真实姓名 | `fldpRMDRcY` | text | |
| 手机号 | `fldNVhDkgh` | text / phone | |
| 微信号 | `fldu5Qqt8S` | text | |
| 微博账号 | `fldKVSWtFR` | text | |
| 抖音账号 | `fldyCSmHWR` | text | |
| 小红书账号 | `fldpoNx0r4` | text | |
| 小宇宙账号 | `fldyfsTsU1` | text | |
| 视频号账号 | `fld73IlJQq` | text | |
| 其他社交账号 | `fldnLX5wMI` | text | |
| 城市 | `fldS89lS1G` | text | |
| 国家 | `fldHjNVlnA` | text | |
| 3句话元故事 | `fldi5HTY1s` | text | |
| 备注 | `fldCD3eRpd` | text | |
| 客户类型 | `fldu7jrWX8` | select | 选项：嘉宾、客户、企业、邀请 |
| 客户标签 | `fldpM5Qnd3` | multi-select | 选项：业务阶段 0-1、业务阶段 1-10、业务阶段 10-100、VIP、IP、副业 |
| 最近跟进时间 | `fldG7qjDc2` | datetime | 格式 yyyy-MM-dd HH:mm |
| 归属人 | `fldwO2hSsR` | link → 团队成员 | |
| 升单人 | `flddF8U4Kl` | link → 团队成员 | |
| 来源渠道 | `fldSlaSag6` | link → 渠道资产 | |
| 所在社群 | `fldRZI8LX3` | link → 渠道资产 | |
| 交付记录 | `fldObh1Vkv` | link → 用户权益明细表 | |
| 客户跟进记录 | `fld3zSW0ZT` | link → 客户跟进记录 | |
| 线索和成交记录 | `fld0U9bVHF` | link → 成交表 | |
| 父记录 | `fldY7t6iyt` | link → 客户名单 | |
| 创建日期 | `fldygfdu3o` | datetime | 通常只读，不手动写入 |
| 归属人_飞书用户 | `fldaT4xyDG` | lookup | **只读，禁止写入** |

## 安全与错误处理

- **新增前必须执行查重流程**：即使用户说「新增」，也要先查重/查相似，确认不重复后方可写入；发现疑似重复时，先展示候选记录并请用户确认是「同一人更新」还是「不同人新增」。
- **更新前必须执行查重流程**：即使用户说「更新某客户」，也要先找到疑似相似记录；匹配到多条或不确定时，必须请用户确认具体 `record_id` 后才能更新。
- **禁止覆盖无关字段**：修改时 `--json` 只包含本次要更新的字段。
- **禁止猜测 ID**：link 字段、select 选项必须来自真实命令返回。
- **scope / 权限错误**：如果 user 身份报授权不足，先转 `lark-shared` 处理认证，不要直接切换 `--as bot`。
- **常见错误码**：
  - `param baseToken is invalid`：检查是否错把 wiki token 或 space_id 当成 base token。
  - `1254045` 字段名不存在：重新 `+field-list`，使用真实字段名或字段 ID。
  - `ignored_fields` / `READONLY`：移除只读字段后重试。

## 输出要求

- 查询结果展示给客户时，核心信息只需包含 `客户昵称`、`真实姓名`、`手机号`、`微信号`；可附带 `record_id` 供后续操作引用，但不要展示无关字段。
- 新增/修改完成后，向用户确认变更内容，并报告最终 `record_id`。
