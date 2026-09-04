---
name: mr-model
description: 「模型先生」+ 任何问题（博主观点/视频检索/评论热词/最近 30 天对个股怎么看）→ 触发本 skill。内部按 11 tool 决策树调用 https://mcp.cesario.top（5 基础 tool + 6 高级 tool：query_video_list / search_videos / query_blogger_opinions / search_video_transcripts / query_comments / query_real_desc_text / query_dimension_levels / query_transcript_keywords / query_aggregated_sentiment / query_creator_meta / query_trending_keywords），用 mcp_tokens Bearer 鉴权。输出两种模式：① 灵活模式（短问答/快查只用 3 段式）② FULL 模式（深度分析输出 `<<<DIA>>>` 11 字段块：主要矛盾/主要方面/看多/看空/多空比/量变质变/现象本质/必然偶然/博主视角/数据视角/交叉关系）+ 合规硬闸（禁个股买卖方向/仓位/价位）。自动联动 ~/.claude/skills/ 下已装行情数据 skill（a-stock-data 等）。需先设置 MR_MCP_TOKEN 环境变量或 ~/.config/mrmodel/token 文件。懒校验、不烧配额、manifest 提示式自更新（jsdelivr CDN 优先 max-age=604800 减少 5min 边缘缓存坑）。
origin: custom
version: 1.3.3
---

# mrmodel-skill — mr-model MCP 调用框架

> **本 skill 不是投资框架，不是博主方法论，不是新分析体系。**
> 它**只是**一个 MCP 调用框架 + 输出规范化层 + 自更新外壳。
> 所有分析逻辑由 MCP 端 `_DIALECTICS_META_PROMPT` (v316 第四层) 注入 + 客户端 LLM 按 `<<<DIA>>>` 契约产出。
> **数据源**：目前仅收录财经博主「模型先生」的 A 股视频数据；后续将接入更多博主，以官网最新公告为准。

## 1. 概述 + 触发词

### 何时激活
- 用户说 **「模型先生」+ 任何问题**（必含「模型先生」4 字触发，LLM 启发式识别）
- 常见别名（命中即触发）：「博主 30 天怎么看 XX」「找视频关于 XX」「MCP 鉴权测试」「查模型先生的视频」

### 适用场景
- ✅ A 股财经观点分析（博主视频/转录/评论）
- ✅ 个股/板块/题材的视频聚合检索
- ✅ 博主最近 X 天对某主题的观点时间线
- ✅ 单视频深度解读（14 字段全量 / 8 维档位 / 转录 5 类分析 / 多空情绪聚合）
- ✅ 平台热词趋势（最近 N 天热词/新词/上升词）
- ✅ 博主 meta（元信息：总视频数/更新频率/影响力）
- ✅ 行情数据（联动 a-stock-data skill）
- ❌ 非 A 股（美股/港股/期货）—— 走 mr-overseas-kline
- ❌ 个股直接买卖建议 —— 合规硬闸硬挡
- ❌ 管理后台操作（封号/改配额等）—— 不在 MCP 能力范围

### 核心约束
- **纯本地工具** — 本 skill 仅本地运行，不落 git 仓
- **只读访问**（token 鉴权）— 只读 + 调 11 tool，无写入/管理能力
- **不入 vault**（vault 是写书素材库，技能工具集职责分离）

### 首次激活引导（v1.2.0 新增）
首次触发本 skill 的会话里，在回答末尾附 1 行提示（仅当次会话首次，不重复刷屏）：
> 还可以问我：① 最近 30 天对 XX（个股）的多空比 ② 最近 7 天平台都在聊什么（热词）③ 某条视频具体讲了什么

### 配额档位速记
- **所有账号**（user / trial / plus / pro）：**人人享 20 quota 终身体验额度**（注册即有，一次性赠送不按月重置，账号状态正常即可，11 tool 全部可用）
- **ProMax**（价格以官网公告为准）：1000 quota / 30 天 + 11 tool 全量
- **admin / sub_admin**：无限（-1）
- 体验额度用尽返 429 `quota_exceeded`：终身体验额度一次性，不按月重置——升级 ProMax 继续用（见 §9.8）

---

## 2. 前置（必读）

### 2.1 token 读取优先级（3 级 fallback）

> token **注册即有**（1 人 1 个，无需申请/创建），完整明文随时在 [mcp-tokens 页](https://mrmodel.cesario.top/mcp-tokens)查看/复制。

```bash
# 优先级 1（推荐）：环境变量
export MR_MCP_TOKEN="mcp_live_<48hex>"

# 优先级 2（fallback）：文件，权限 600
mkdir -p ~/.config/mrmodel
echo -n "mcp_live_<48hex>" > ~/.config/mrmodel/token
chmod 600 ~/.config/mrmodel/token

# 优先级 3（都不存在）：报错兜底
# → 告诉用户 token 注册即有，去 https://mrmodel.cesario.top/mcp-tokens 查看/复制完整 token
```

### 2.2 token 格式校验（启动时轻量预检，0 配额成本）

- 前缀：`mcp_live_`（**9 字符**：`m-c-p-_-l-i-v-e-_`）
- 后跟：**48 个十六进制字符**（`[0-9a-f]{48}`，sha256 派生生成，服务端只存哈希不存明文）
- 总长：**57 字符**
- **格式正确 → 静默通过，第一次实际调用时才连 MCP 鉴权**
- **格式错误 → 立即报错 "token 格式不合法，应为 mcp_live_<48hex> 共 57 字符"**

### 2.3 测试连通性命令（可选，0 配额成本）

> ⚠️ **请求头缺一不可（实测 2026-09-04）**：MCP 端套了 Cloudflare 代理，`User-Agent` 缺失或不像浏览器 → **403（error 1010）**；`Accept` 不含 `text/event-stream` → **406**。下面的示例已带全，直接复制可用。

```bash
# 11 tool 清单测试（tools/list 不烧配额）
curl -s -X POST https://mcp.cesario.top/mcp \
  -H "Authorization: Bearer $MR_MCP_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "User-Agent: Mozilla/5.0" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

**响应是 SSE 流格式**，11 个 tool 名在 `data:` 行的 JSON 里（`result.tools[].name`）：

```
event: message
data: {"jsonrpc":"2.0","id":1,"result":{"tools":[{"name":"query_video_list",...}, ...]}}
```

```bash
# 一行提取 11 个 tool 名（验证鉴权通过）
curl -s -X POST https://mcp.cesario.top/mcp \
  -H "Authorization: Bearer $MR_MCP_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "User-Agent: Mozilla/5.0" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' \
  | grep '^data:' | sed 's/^data: //' | python3 -c "import json,sys; print([t['name'] for t in json.load(sys.stdin)['result']['tools']])"
```

**传输层语义（实测 2026-09-04）**：
- **无状态**：服务端不返回 `Mcp-Session-Id`，**不需要** initialize 握手，每个 POST 独立携带 token 即可
- 直接 POST `tools/call` 可行（跳过 initialize + notifications/initialized 照样成功）
- 每次请求都必须带上面 4 个头（Authorization / Content-Type / Accept / User-Agent）

---

## 3. 11 tool 调用决策树

### 3.1 决策树（用户问句 → 调哪个 tool，5 基础 + 6 高级）

```
用户问题
├─ 包含"最近"/"最新"/"今天"/"昨天" + "视频"？
│   └─ YES → query_video_list(page=1, page_size=20)  ← 默认 20 条
├─ 包含"博主最近"/"30 天"/"X 天" + "怎么看" + 个股/板块名？
│   └─ YES → query_blogger_opinions(keyword=..., date_from=-30d, date_to=今天, limit=20)
├─ 包含"转录里"/"说过"/"提过" + 关键词？
│   └─ YES → search_video_transcripts(keyword=..., limit=20)
├─ 包含"评论"/"评论区"/"热不热"？
│   └─ YES → 先 query_video_list 拿最新视频 → 读 dict.aweme_id → query_comments(aweme_id=...)
│            （query_comments 返的是聚合统计视图：total_comments/avg_digg/top_keywords，不是评论 list）
├─ 包含具体关键词（个股名/板块名/概念名）但无时间限定？
│   └─ YES → search_videos(query=..., page=1, page_size=20)  ← 模糊兜底
│
├─ 已知 aweme_id，要单视频 14 字段完整元信息（desc_text + 8 维 + analysis_framework）？
│   └─ YES → query_real_desc_text(aweme_id=...)  ← 高级 tool
├─ 已知 aweme_id，要看 8 维档位（0/1/2 三档命中）？
│   └─ YES → query_dimension_levels(aweme_id=...)  ← 高级 tool
├─ 已知 aweme_id，要转录 5 类分析（词频/NER/词性/关键句/摘要 prompt）？
│   └─ YES → query_transcript_keywords(aweme_id=...)  ← 高级 tool
├─ 含"多空"/"看多看空"/"情绪"/"拐点" + 关键词？
│   └─ YES → query_aggregated_sentiment(keyword=..., granularity=weekly|monthly)  ← 高级 tool
│            （⚠️ 0 命中时返空 weekly_distribution={} 无 _hint 提示 → 降级调
│              query_blogger_opinions 拉原始视频，由 LLM 自行归纳多空）
├─ 含"博主"/"靠不靠谱"/"影响力"/"活跃度"/"更新频率"？
│   └─ YES → query_creator_meta(sec_uid=...)  ← 高级 tool（默认唯一博主"模型先生"）
└─ 含"最近热词"/"平台上在聊什么"/"新词"/"上升词"？
    └─ YES → query_trending_keywords(days=7, top_n=50, sort_by=videos)  ← 高级 tool
```

### 3.2 11 tool 输入参数速查

#### 5 基础 tool

> **配额公式**：`cost = ⌈base + 行数 × per⌉ quota`（向上取整，防拖库；非 list 返 base 单次）
> **单位**：**quota**（配额点；ProMax 享 1000 quota / 30 天滚动窗口，其余档位人人享 20 quota 终身体验额度，一次性不按月重置）

| Tool | 必填 | 关键可选 | 默认值 | 配额成本 (quota) | 返回类型 |
|------|------|----------|--------|------------------|----------|
| `query_video_list` | — | `page`, `page_size` (≤20) | page=1, page_size=20 | base=1, per=0.1×N（page_size=20 → 3） | **list[dict]**（单条 video 全字段） |
| `search_videos` | `query` | `page`, `page_size` (≤20) | page=1, page_size=20 | base=1, per=0.1×N（page_size=20 → 3） | **list[dict]**（0 命中时返 `_hint` dict） |
| `query_blogger_opinions` | `keyword` (≥2字) | `date_from`, `date_to`, `limit` (1-20) | limit=20 | base=2, per=0.1×N（limit=20 → 4） | **list[dict]**（0 命中时 `_hint.reason=no_match`） |
| `search_video_transcripts` | `keyword` | `limit` (1-20) | limit=20 | base=2, per=0.05×N（limit=20 → 3） | **list[dict]**（含转录 snippet ≤65 字） |
| `query_comments` | `aweme_id` | — | — | 1（dict 聚合，per_row 不计） | **dict 聚合**（total/avg_digg/max_digg/top_keywords） |

#### 6 高级 tool（v1.1.0 新增）

> **配额公式**：`cost = base quota`（高级 tool 全是 dict 返回，不走 per_row，base 已含计算成本）

| Tool | 必填 | 关键可选 | 默认值 | 配额成本 (quota) | 返回类型 |
|------|------|----------|--------|------------------|----------|
| `query_real_desc_text` | `aweme_id` (18-20位) | — | — | 1 | **dict**（14 字段：VIDEO_LIST_ALLOWLIST + dialectics_tags + framework_dimensions + analysis_framework） |
| `query_dimension_levels` | `aweme_id` | — | — | 1 | **dict**（8 维每维 level 0/1/2 + label 翻译 + 数据源 + 分析步骤） |
| `query_transcript_keywords` | `aweme_id` | — | — | 2 | **dict**（5 类：词频 Top50 + NER + 词性 + 关键句 Top5 + 摘要 prompt） |
| `query_aggregated_sentiment` | `keyword` (≥2字) | `date_from`, `date_to`, `granularity` (weekly\|monthly) | granularity=weekly | 2 | **dict**（周/月桶计数 + 拐点；⚠️ 不传 date_from/date_to 默认全量时间窗，实测常返空结果，建议显式传时间窗） |
| `query_creator_meta` | `sec_uid` (可选) | — | 当前唯一博主"模型先生" | 1 | **dict**（stats 10 字段：视频数/点赞/评论/分享/更新频率等） |
| `query_trending_keywords` | — | `days` (1-30), `top_n` (10-100), `sort_by` (videos\|digg\|comment) | days=7, top_n=50, sort_by=videos | 2 | **dict**（窗口热词 + 新词 + 上升词） |

**关键差异（实测 2026-09-04）**：
- ❌ 不是「list 包 dict」形态
- ✅ 全部 dict 形态（page_size=1 单条 / >1 时 list 包 dict）
- ✅ 顶级字段直接是 video 数据 + `_meta`（quota_cost/remaining） + `_tx_id`（uuid4）
- ✅ 0 命中时 query_blogger_opinions / search_videos 返 `_hint` dict 替代空结果（**query_aggregated_sentiment 例外：0 命中返空 `weekly_distribution: {}`，无 _hint**）
- ✅ 6 高级 tool 都是单条 dict 返回，aweme_id/keyword 是必填

**⭐ JSON-RPC content 多 item 解析（必读，最容易踩的坑）**：

服务端（FastMCP）把 **list[dict] 类型的 tool 返回值拆成 N 条独立的 content item**——每条视频一个 text item，**没有** structuredContent。客户端拿到响应后必须**遍历 `result.content[]` 逐条 `json.loads`**：

```
query_video_list(page_size=20) 的响应结构：
result.content = [
  {"type": "text", "text": "{视频1的完整JSON}"},   ← content[0]
  {"type": "text", "text": "{视频2的完整JSON}"},   ← content[1]
  ...
  {"type": "text", "text": "{视频20的完整JSON}"}   ← content[19]
]
```

```python
# 错误写法（拿到空/解析失败）：json.loads(result.content[0].text) 只拿到第 1 条
# 正确写法：遍历全部 content item 逐条解析
videos = [json.loads(item.text) for item in response["result"]["content"] if item["type"] == "text"]
```

- 单 dict 返回的 tool（query_comments / 全部 6 高级 tool）→ content 只有 **1 条** item，`json.loads(content[0].text)` 即可
- 0 命中返 `_hint` 时 → content 也只有 1 条 item，text 里是 `{"_hint": {...}, "_tx_id": ...}`
- 判断命中条数用 `len(result.content)`，**不要**在 text 里数

### 3.3 配额保护策略（KISS：宁可少调不烧配额）

> 单位：**quota**（配额点；ProMax 1000 / 30 天滚动窗口，其余档位 20 quota 终身体验）

1. **默认 page_size=20**（query_video_list 单次 cost=3 quota），超 20 提示用户"是否需要翻第 2 页"（page=2 需用户显式确认）
2. **search_videos page_size=20**（cost=3 quota）+ **query_blogger_opinions limit=20**（cost=4 quota，默认足够覆盖博主典型 7-30 天观点；控成本可手动降到 limit=10 → cost=3）
3. **search_video_transcripts limit=20**（cost=3 quota，snippet ≤65 字 × 20 = 约 1300 字；轻量快查可手动降到 limit=5 → cost=2，token 更经济）
4. **query_comments 单次 1 个 aweme_id**（聚合 dict 统计，cost=1 quota）
5. **高级 tool base 1-2 quota**（query_transcript_keywords / query_aggregated_sentiment / query_trending_keywords cost=2 quota，含 jieba/NER/聚合计算；query_real_desc_text / query_dimension_levels / query_creator_meta cost=1 quota）
6. **不级联调用**：拿不到结果就告诉用户，不无限重试
7. **配额透明（v1.2.0 新增）**：每次 MCP 调用成功后，展示一行「本次消耗 X quota」（读返回的 `_meta.quota_cost`）。⚠️ **`_meta.quota_remaining` 在服务端当前版本不可靠（实测恒定不递减），不要向用户展示或据此判断余量**——余量以官网 mcp-tokens 页显示为准，避免"明明还有额度却以为用尽"的误导

### 3.4 决策树禁忌

- **不调 query_video_list 两次**（避免浪费）— 第一次拿不到想要的关键词结果时，转 search_videos
- **不重复调 query_blogger_opinions** 同一 keyword 同一时间窗（让 LLM 缓存前次结果）
- **不调 query_comments 整列表**（只对最新 1-2 个视频取评论，节省配额）
- **不并行调多个高级 tool**（cost=2 叠加爆配额，串行调用更好）
- **已知 aweme_id 时优先 query_real_desc_text**（避免先 query_video_list 拿 id 再调的中间步骤）

### 3.5 0 命中处理（`_hint` 字段识别 + sentiment 空结果）

MCP 端 v326 M3 治本：**query_blogger_opinions / search_videos** 0 命中时返 `_hint` 字典替代 list 包空 dict：

```json
{
  "_hint": {
    "reason": "no_match",
    "tool": "query_blogger_opinions",
    "suggestion": "尝试简化关键词 / 扩时间窗口 / 检查拼写"
  },
  "_tx_id": "uuid4-xxxx"
}
```

LLM 看到 `_hint.reason == "no_match"` → 提示用户按 suggestion 调整查询，**不要重试相同参数**。

⚠️ **query_aggregated_sentiment 例外（实测 2026-09-04）**：0 命中时**不返 `_hint`**，而是返回 `total_videos: 0` + 空 `weekly_distribution: {}` 的静默空结果。LLM 拿到空 sentiment 后的正确降级路径：

1. 放宽/去掉 `date_from`/`date_to`（不传 = 全量时间窗）重试 1 次
2. 仍为空 → 降级调 `query_blogger_opinions(keyword=...)` 拉原始视频列表，由 LLM 自行归纳多空观点
3. 告诉用户"该关键词暂无聚合多空数据"，不要编造多空比

### 3.6 高级 tool 何时用（决策细化）

| 场景 | 推荐 tool | 原因 |
|------|----------|------|
| 用户问"这条视频说的啥"（已知 aweme_id） | `query_real_desc_text` | 14 字段全量，比 query_video_list 字段更多（含 analysis_framework） |
| 用户问"这视频哪几维命中" | `query_dimension_levels` | 带 level 0/1/2 档位 + label 翻译，LLM 不用自己算分 |
| 用户问"这视频转录讲了哪些标的" | `query_transcript_keywords` | 5 类分析（含 NER 实体 + 摘要 prompt），省 LLM 自己跑 NER |
| 用户问"30 天对中际旭创多空比" | `query_aggregated_sentiment` | 返周/月桶计数 + 拐点省归纳；⚠️ 空结果无 _hint，需降级 query_blogger_opinions（见 §3.5） |
| 用户问"模型先生活跃度怎么样" | `query_creator_meta` | 一次性返 stats 10 字段（更新频率/影响力），不烧多调配额 |
| 用户问"最近一周平台在聊啥" | `query_trending_keywords` | days=7 默认 + top_n=50，3 类词（top/new/rising）一次到位 |
| 用户问"XX 最近 30 天怎么说" | `query_blogger_opinions`（基础） | 时间线聚合 + dialectics_tags，够用 |

---

## 4. 辩证法分析模板（灵活 + FULL 双模式）

> **复用 MCP 端 v316 第四层**：`_DIALECTICS_META_PROMPT` (mcp_server.py:257-267) + `_RISK_VOCABULARY` (mcp_server.py:270-279) + `_TIME_HORIZON_TEMPLATE` (mcp_server.py:282-298)
>
> **本节是 system prompt 拼装范本**，LLM 收到 MCP 透出的 `analysis_framework` 字段后，按以下结构组织输出。

### 4.0 模式选择（v1.1.0 新增，避免过度输出烧 token）

| 模式 | 触发场景 | 输出要求 | token 量级 |
|------|---------|---------|-----------|
| **灵活模式** | 快查 / 短问答 / 单视频快解 / "简要说说" / 闲聊 | 只用 3 段式（① 正反论据 ② 转化条件 ③ 分时段判断） | ~300-500 字 |
| **FULL 模式** | 深度分析 / 持仓决策辅助 / 学术性研究 / 用户明确说"详细分析" | 3 段式 + `<<<DIA>>>` 11 字段块 | ~800-1500 字 |

**判别口诀**（LLM 启发式）：
- 用户说"详细 / 完整 / 全面 / 深度 / 系统 / 多维度" → **FULL 模式**
- 用户说"简要 / 简略 / 快速 / 简答 / 一句话" → **灵活模式**
- 用户没说但 token 预算紧（聊天/移动端/embedding 长上下文）→ **灵活模式优先**
- 默认未指定 → 走 **灵活模式**（KISS，少烧 token，v1.1.0 默认降级）

### 4.1 system prompt 拼装（4 件套，灵活/FULL 模式都共用）

LLM 调 MCP 拿到 video 数据后，**把以下 4 块拼进 system prompt**：

```
# 1. 辩证元框架（来自 MCP 透出 analysis_framework.dialectics_meta_prompt）
{_DIALECTICS_META_PROMPT}

# 2. 当前视频所属维度（来自 MCP 透出 dialectics_tags + framework_dimensions）
{dialectics_tags} + 各维度 description + suggested_data_sources + analysis_steps

# 3. 风险信号词表（来自 MCP 透出 analysis_framework.risk_vocabulary）
{_RISK_VOCABULARY 8 词}

# 4. 输出要求（合规硬闸）
严格按「正反论据对照→转化条件→分时段判断」结构输出
禁用具体买入/卖出点位、仓位比例、目标价、止损价
合规：末尾原样追加 §4.6 合规声明常量（逐字粘贴，禁改写/缩写/删除）
（FULL 模式加 1 行：末尾追加 `<<<DIA>>>` 11 字段块）
```

### 4.2 输出结构

#### 灵活模式（v1.1.0 新增，token 友好）

```markdown
## ① 正反论据对照表
| 看多 | 看空 |
|------|------|
| 论据1 | 论据1 |
| 论据2 | 论据2 |

## ② 转化条件清单
- 看多失效条件：...
- 看空失效条件：...

## ③ 分时段判断
- 短期（1-4周）：...
- 中期（1-6月）：...
- 长期（半年+）：...
```

#### FULL 模式（3 段式 + `<<<DIA>>>` 11 字段块）

```markdown
## ① 正反论据对照表
| 看多 | 看空 |
|------|------|
| 论据1 | 论据1 |
| 论据2 | 论据2 |

## ② 转化条件清单
- 看多失效条件：...
- 看空失效条件：...

## ③ 分时段判断
- 短期（1-4周）：技术面主导，...
- 中期（1-6月）：基本面+景气度，...
- 长期（半年+）：产业逻辑，...

<<<DIA>>>
主要矛盾|...
主要方面|...
看多|...
看空|...
多空比|...
量变质变|...
现象本质|...
必然偶然|...
博主视角|...
数据视角|...
交叉关系|...
<<<END>>>
```

### 4.3 `<<<DIA>>>` 11 字段语义速查

| 字段 | 含义 | 素材来源 |
|------|------|----------|
| 主要矛盾 | 当前博弈的核心张力点 | LLM 归纳（MCP 不直接返） |
| 主要方面 | 矛盾的主导面（看多/看空/震荡） | LLM 归纳 |
| 看多 | 3-5 条核心多头论据 | 视频 desc_text + 转录 snippet |
| 看空 | 3-5 条核心空头论据 | 同上 |
| 多空比 | 多空力量量化（如 6:4） | LLM 自行量化（MCP 不给数字） |
| 量变质变 | 当前所处阶段（积累期/转折期/扩散期） | LLM 归纳 |
| 现象本质 | 表面现象 vs 底层本质的区分 | LLM 归纳 |
| 必然偶然 | 大趋势必然性 vs 短期偶然性分离 | LLM 归纳 |
| 博主视角 | 博主的核心观点（**改称"博主"不直呼名**） | query_blogger_opinions 聚合 |
| 数据视角 | MCP 透出 8 维框架维度的命中 | framework_dimensions |
| 交叉关系 | 博主视角 ↔ 数据视角 的一致/分歧点 | LLM 对话式归纳 |

### 4.4 风险信号词（命中需特别警惕）

`_RISK_VOCABULARY` 8 词：
- 情绪过热 / 分化失序 / 共识明牌化 / 量价背离
- 高位放量滞涨 / 均线死叉确认 / 支撑位失守 / 业绩不及预期

LLM 输出时若命中，**该词单独标红或加粗**（前端渲染层判断）。

### 4.5 三时段模板

| 时段 | 周期 | 主导 | 关键问题 |
|------|------|------|----------|
| 短期 | 1-4 周 | 技术面 | 关键支撑位？短期量能？有无突破/跌破信号？ |
| 中期 | 1-6 月 | 基本面+行业景气 | 业绩能否支撑估值？有无行业/政策催化？主力资金方向？ |
| 长期 | 半年+ | 宏观+产业逻辑 | 产业大逻辑成立？竞争格局改善？技术迭代颠覆？ |

### 4.6 合规硬闸（必须落实，LLM 输出前自检，v1.2.0 强化）

❌ **禁用字眼**（命中 → 改写为通用方法论提示）：

操作指令类：
- 买入 / 卖出 / 加仓 / 减仓 / 止损 / 止盈
- 建仓 / 补仓 / 低吸 / 追入 / 右侧追 / 介入 / 目标价

收益承诺类（v1.2.0 新增——荐股诈骗核心特征，监管红线，命中即改写中性表述，零豁免）：
- 稳赚 / 稳赚不赔 / 必涨 / 必跌 / 无风险 / 零风险 / 百分百 / 百分之百 / 包赚 / 保底 / 抄底逃顶

❌ **禁用内容**：
- 具体仓位比例（如"建议 30% 仓位"）
- 具体点位（如"在 45.20 元买入"）
- 具体目标价/止损价
- **个性化建议（v1.2.0 新增）**：禁止基于用户持仓/风险偏好/资金量给建议（如"你适合加仓""你的情况可以买"）——只输出"通用方法论 + 数据事实"，不做个人投顾

✅ **允许内容**：
- 通用投资方法论（"风控需关注 X 条件"）
- 转化条件失效提示（"看多逻辑失效时需重新评估"）
- 数据/事实陈述（"当前 PE 分位 80%"）

**合规声明常量（v1.2.0 钉死）**：

所有输出（灵活模式 + FULL 模式，无一例外）末尾**原样追加**以下常量，**逐字粘贴，禁止改写/缩写/删除**：

```
⚠️ 声明：本内容由 AI 聚合生成，非持牌证券投资顾问意见，不构成任何投资建议；数据来自第三方，可能存在延迟或偏差，请以官方信息为准；投资有风险，请自行决策并承担风险。
```

**LLM 输出后自检清单**（任一失败 → 改写重查）：
1. 全文扫描禁用字眼（操作指令类 + 收益承诺类）
2. 检查是否含具体数字仓位/点位
3. 检查是否含个性化建议（基于"你的持仓/偏好/资金量"）
4. 末尾是否原样含合规声明常量（缺 → 补）

### 4.7 输出对齐要点（不可破坏）

平台输出规范（与官网 AI 分析对齐）：
- **隐私红线 P0**：不透露 GLM/Claude 等模型名（用"技术实现细节不便透露"）
- **禁提数据源品牌**：涨乐/华泰/腾讯/东方财富/同花顺/任何 skill/API 统一用"本平台数据"
- **"凡指代博主都改博主"**（博主表达方式约定）
- **加粗 / 表格 / 固定结尾声明**：必须落实

---

## 5. 行情 skill 整合

### 5.1 静态白名单（已知 1 个 skill）

```yaml
known_market_skills:
  - name: a-stock-data
    description: "A股全栈数据工具包 — 行情(mootdx+腾讯+百度K线)、研报(东财+同花顺+iwencai)、信号(同花顺热点+北向+百度PAE+龙虎榜+解禁+行业)、资金面(融资融券+大宗交易+股东户数+分红+资金流)、新闻(东财+财联社)、基础数据(mootdx财务/F10+东财+新浪三表)、公告(巨潮)七层数据源"
    path: ~/.claude/skills/a-stock-data/SKILL.md
    version: 3.0
    trigger_keywords: [PE, 估值, K线, 行情, 研报, 龙虎榜, 北向, 资金, 财务, 公告, 换手率, 市值]
```

### 5.2 启动时扫描逻辑（LLM 启发式行为）

用户说"模型先生 + 股票代码/估值/PE/财务"时：

1. **扫描** `~/.claude/skills/*/SKILL.md`
2. **匹配 trigger_keywords**（中文：行情/估值/K线/研报/财务/公告/龙虎榜/北向/资金；英文：stock/finance/kline/quote/valuation/fundamental/market）
3. **命中** → 在 LLM 上下文里**注入**该 skill 存在声明（仅 skill 名 + 触发词 + 描述首行，**不复制全文**）
4. **LLM 决定何时调用**（不是 skill 自动调）：
   - 查"中际旭创" → 先 `query_blogger_opinions` 拿博主观点 → 发现需 PE → 调 a-stock-data 的 `tencent_quote` → 综合输出
5. **未命中** → 告诉用户"未检测到行情 skill，建议安装 a-stock-data"

### 5.3 告知 LLM 怎么用（写到 system prompt）

```
MCP 11 tool 不提供实时行情数据。
- 视频/博主/转录/评论维度 → 调 MCP 11 tool
- 行情/估值/财务/资金面 → 联动 a-stock-data skill
两者互补不重叠，组合使用得到完整分析。
```

### 5.4 行情 skill 不存在的兜底

若用户没装 a-stock-data：
- 仍返回博主观点 + 框架维度（不报错）
- 在输出末尾加一行"（未装行情 skill，建议安装 a-stock-data 获取实时 PE/K线/资金面数据）"

---

## 6. 输出范本

### 6.1 成功范本（典型用法，灵活模式）

**用户**：「模型先生，30 天对中际旭创怎么看？」

**LLM 行为**（v1.1.0 灵活模式）：
1. 触发 skill 加载
2. 匹配决策树 → `query_blogger_opinions(keyword="中际旭创", date_from=-30d, date_to=今天, limit=20)`
3. 拼 4 件套 system prompt（含 `_DIALECTICS_META_PROMPT` + 8 维 + 风险词 + 合规硬闸）
4. 输出**灵活模式** 3 段式（用户没说"详细"，默认走灵活模式省 token）

```markdown
## ① 正反论据对照表
| 看多 | 看空 |
|------|------|
| 北美大客户 800G/1.6T 需求持续放量 | 头部光模块厂订单增速 Q3 边际放缓 |
| 国产替代逻辑（硅光 CPO 路线） | 板块 PE 分位已达 80% 历史高位 |

## ② 转化条件清单
- 看多失效：北美云厂商资本开支同比转负 / CPO 量产延迟至 2027
- 看空失效：800G 出货量超预期 / 1.6T 招标提前到 Q4

## ③ 分时段判断
- 短期（1-4周）：跟随板块情绪，量能若缩则需警惕
- 中期（1-6月）：看 800G/1.6T 招标节奏
- 长期（半年+）：硅光 CPO 产业逻辑兑现进度

⚠️ 声明：本内容由 AI 聚合生成，非持牌证券投资顾问意见，不构成任何投资建议；数据来自第三方，可能存在延迟或偏差，请以官方信息为准；投资有风险，请自行决策并承担风险。
```

### 6.2 高级 tool 范本（v1.1.0 新增）

#### 6.2.1 query_real_desc_text（单视频 14 字段全量）

**用户**：「模型先生，这条视频（aweme_id=7677520767986234289）讲的啥？」

**LLM 行为**：直接调 `query_real_desc_text(aweme_id="7677520767986234289")`（aweme_id 传 **string 带引号**，省去先 query_video_list 拿 id 的中间步）

**返回 dict**（14 字段）：
- `aweme_id` / `desc_text` / `create_time` / `create_time_str`
- `duration` / `statistics` / `tags` / `content_type`
- `dialectics_tags` / `framework_dimensions` / `analysis_framework`
- `author_nickname` / `author_sec_uid` / `_desc_note`

#### 6.2.2 query_aggregated_sentiment（多空情绪聚合 + 拐点）

**用户**：「模型先生，30 天对中际旭创多空比多少？」

**LLM 行为**：调 `query_aggregated_sentiment(keyword="中际旭创", date_from="2026-08-05", date_to="2026-09-04", granularity="weekly")`（**显式传时间窗**，不传默认全量易空结果）

**返回 dict 实测字段（2026-09-04）**：
- `keyword` / `granularity` / `date_from` / `date_to` / `total_videos`
- `neutral_count`（中性计数）
- `weekly_distribution`（桶结构：`{"2026-W35": {"neutral": 0, "videos": 1}}`；monthly 时为 `monthly_distribution`）
- `trend_inflection_points`（拐点列表）
- ⚠️ **当前版本不返回** `bull_count` / `bear_count` / `bull_bear_ratio` / `top_bull_quotes` / `top_bear_quotes` 字段（多空分桶依赖服务端词典命中，未命中视频只进 `videos` 计数）；多空对比由 LLM 基于视频内容自行归纳

**LLM 输出要点**：
- 按桶计数描述趋势走向（如"W32 中性 1 条 → W35 多头为主"），拐点列表如实转述
- 0 命中（空 `weekly_distribution`）→ 走 §3.5 降级路径
- ⚠️ **不替用户做对错判断**，只罗列数据 + 拐点

#### 6.2.3 query_dimension_levels（8 维档位 0/1/2）

**用户**：「模型先生，这条视频哪几维命中了？」

**LLM 行为**：调 `query_dimension_levels(aweme_id="...")`

**返回 dict**：`dimension_scores: { 估值类: {score, description, suggested_data_sources, analysis_steps, level, label}, 趋势类: {...}, ... 8 维 }`

**level 含义**：
- `0` = 未命中（score<0.3）
- `1` = 中性（0.3 ≤ score < 0.8）
- `2` = 强信号（score ≥ 0.8 或 `['综合']` 弱信号下 0.5）

#### 6.2.4 query_transcript_keywords（转录 5 类分析）

**用户**：「模型先生，这条视频转录里提到了哪些标的？」

**LLM 行为**：调 `query_transcript_keywords(aweme_id="...")`（**配额 cost=2**，含 jieba+NER）

**返回 dict**（5 类）：
- `word_freq`（jieba TF-IDF Top50）
- `entities`（NER 实体识别：stock / concept / kol 3 类，dict_match v1）
- `pos_distribution`（词性分布 dict）
- `key_sentences`（关键句 Top5）
- `transcript_summary_prompt`（拼好给客户端 LLM 加工的提示字符串，**不替 LLM 做 NLU**）

#### 6.2.5 query_creator_meta（博主元信息）

**用户**：「模型先生，模型先生靠不靠谱？」

**LLM 行为**：调 `query_creator_meta()`（默认唯一博主"模型先生"）

**返回 dict 关键字段（实测 2026-09-04）**：
- `sec_uid` / `author_nickname` / `stats`
  - stats: `total_videos` / `videos_last_30d` / `videos_last_7d`
  - `total_digg` / `total_comment` / `total_share`
  - `avg_duration_sec` / `max_gap_days`
  - `first_video_at` / `last_video_at`（**epoch 秒**，非日期字符串）
- ⚠️ 当前版本**不透出** `activity_score` 字段（活跃度评分待主人拍权重后上线），判断活跃度用 `videos_last_30d` / `max_gap_days` 自行归纳

#### 6.2.6 query_trending_keywords（平台热词）

**用户**：「模型先生，最近一周平台在聊什么热词？」

**LLM 行为**：调 `query_trending_keywords(days=7, top_n=50, sort_by=videos)`

**返回 dict 关键字段（实测 2026-09-04）**：
- `window: {days, from, to}`
- `top_keywords`（当前窗口热词，元素为 `{word, videos, total_digg, total_comment}` dict）
- `new_keywords`（本窗口新出现词，元素为 `{word, videos, total_digg}` dict）
- `rising_keywords`（环比增长 > 1.5，元素为 `{word, current_videos, prev_videos, growth_ratio}` dict）

#### 6.2.7 范本：3 tool 组合（v1.1.0 推荐范式）

**用户**：「模型先生，中际旭创最近 30 天怎么走 + 8 维 + 拐点？」

**LLM 行为**（**串行不并行**，省配额）：
1. `query_aggregated_sentiment(keyword="中际旭创", date_from=-30d)` → 多空比 + 拐点
2. 取最热 1 个 video 的 aweme_id → `query_dimension_levels(aweme_id=...)` → 8 维档位
3. （可选）`query_transcript_keywords(aweme_id=...)` → 转录 5 类分析（cost=2，看配额）

**总配额成本**：2 (sentiment) + 1 (dimensions) = 3 quota（可选加转录分析 +2），**比 5 基础 tool 的 N 次联调省得多**

### 6.3 0 命中范本

**用户**：「模型先生，30 天对 XX（无此标的视频）怎么看？」

**MCP 返**：
```json
{"_hint": {"reason": "no_match", "tool": "query_blogger_opinions", "suggestion": "尝试简化关键词 / 扩时间窗口 / 检查拼写"}}
```

**LLM 输出**：
> 未找到 30 天内关于"XX"的视频。建议：① 简化关键词 ② 扩时间窗口到 90 天 ③ 检查拼写。
> tx_id: `uuid-xxx`（可在官网 mcp-tokens 页查用量）

### 6.4 配额超限范本

**MCP 返 429**：`quota_exceeded`，`Retry-After: 86400`

**LLM 输出**：
> 您的 20 quota 免费体验额度已用完（终身一次性，不按月重置）。
> 升级 ProMax（1000 quota / 30 天）可继续使用：登录官网 → 会员中心；ProMax 本期用尽则等本期结束自动重置。

### 6.5 鉴权失败范本

**MCP 返 401**：`invalid_token`

**LLM 输出**：
> token 无效或已重置，请检查：
> 1. 确认 `MR_MCP_TOKEN` 环境变量或 `~/.config/mrmodel/token` 文件已正确配置
> 2. 确认 token 格式为 `mcp_live_<48hex>`（共 57 字符）
> 3. 完整 token 注册即有，随时去 https://mrmodel.cesario.top/mcp-tokens 查看/复制；如怀疑泄露，在该页点「重置」后换用新 token

---

## 7. 自更新（manifest 提示式，jsdelivr 优先）

### 7.1 manifest.json 字段

```json
{
  "name": "mr-model",
  "version": "1.3.3",
  "min_mcp_server_version": "326",
  "skill_md_sha256": "<sha256-of-this-file>",
  "skill_md_url": "https://cdn.jsdelivr.net/gh/Cesario-Lzc/M-Model@main/SKILL.md",
  "manifest_url": "https://cdn.jsdelivr.net/gh/Cesario-Lzc/M-Model@main/manifest.json",
  "updated_at": "2026-09-04T...Z",
  "changelog": "v1.3.3: 生产实测校准 — 连通性 curl 补 CF 必需头(UA/Accept) + SSE data: 解析, content[] 多 item 解析教学, 参数默认值对齐 server schema(全 20), sentiment/creator_meta 幻影字段清理, 附录 A 全量真机重测(2026-09-04)"
}
```

**URL 优先级（v1.1.0 重要更新）**：
- ✅ **首选 jsdelivr CDN**（`cdn.jsdelivr.net/gh/Cesario-Lzc/M-Model@main/...`，max-age=604800=7 天，默认走 jsdelivr）
- ⚠️ **raw.githubusercontent 备选**（max-age=300=5min 边缘缓存坑，访问频繁会触发 GitHub 限流）
- 📌 当前 install 脚本默认走 jsdelivr（commit 58ed2aa）

### 7.2 检查流程（首次调用时 lazy 触发）

LLM 第一次调 MCP tool 之前：

1. 读本地 `manifest.json` + `mtime`（文件最后修改时间）
2. **mtime < 1 小时 → 跳过检查**（避免短时间多次拉取）
3. 否则发 GET 到 `manifest_url`（**不**烧 MCP 配额，走静态资源）
4. 对比 `skill_md_sha256`：
   - **一致** → 继续，**不**更新 mtime
   - **不一致** → **提示用户**："发现新版本 v1.x.x，changelog 摘要：...，是否更新？"
     - 选是 → 下载新 SKILL.md 覆盖本地 + 写新 manifest.json
     - 选否 → 跳过（KISS 尊重用户选择）
5. 检查 `min_mcp_server_version`：如果本地 MCP 端 < 该版本 → 警告"部分功能可能不可用"

### 7.3 降级策略

- **远端 manifest 拉不到**（404 / 5xx）→ 静默继续用本地版本（不阻塞用户）
- **沙盒/离线场景** → 拉不到 = 静默继续
- **网络超时**（>5s）→ 跳过本次检查

### 7.4 手动更新

```bash
# 强制重拉
rm ~/.claude/skills/mr-model/manifest.json
# 下次调用时自动拉新
```

---

## 8. 错误码处理（全档映射，v1.1.0 加 ProMax 倒计时）

### 8.1 401 未鉴权

| 错误码 | 原因 | 兜底话术 |
|--------|------|----------|
| `missing_bearer` | 漏了 Authorization 头 | 「还没有配置 token 哦～token 注册即有（1 人 1 个），登录 https://mrmodel.cesario.top/mcp-tokens 查看/复制，写入 MR_MCP_TOKEN 环境变量或 ~/.config/mrmodel/token 文件即可」 |
| `invalid_token` | token_hash 不匹配 / status≠active | 「这个 token 无效或已重置～完整明文随时在 https://mrmodel.cesario.top/mcp-tokens 查看/复制（注册即有）；如确认 token 正确仍报错，欢迎登录官网联系我们处理」 |
| `expired` | expires_at < now | 「token 已过期啦～在 https://mrmodel.cesario.top/mcp-tokens 点「重置」换新即可，同一入口随时可查明文；如有疑问欢迎官网联系我们」 |

### 8.2 403 已鉴权但禁止（账号状态异常）

| 错误码 | 原因 | 兜底话术 |
|--------|------|----------|
| `account_disabled` | 账号已禁用 | 「您的账号当前处于禁用状态，暂时无法调用～如认为是误判，欢迎登录官网联系我们核实处理」 |
| `account_banned` | 账号被封禁 | 「账号封禁中，封禁结束后会自动恢复～如有疑问欢迎登录官网联系我们」 |
| `mcp_not_enabled` | 当前账号未开放 MCP（罕见；正常注册账号人人享 20 quota 终身体验） | 「当前账号暂未开放 MCP 调用～正常注册账号均享 20 quota 终身体验额度，如确认账号状态正常仍报此错，请登录官网联系开发者处理；如需大配额可升级 ProMax（见 §9.8）」 |

**契约要点**（2026-09-03 v1.3.2 更新）：
- **人人保底 20（终身）**：所有账号状态正常的用户均享 20 quota 终身体验额度，一次性赠送不按月重置（11 tool 全部可用）
- **ProMax** 1000 quota / 30 天滚动窗口（本期用尽等本期结束自动重置）；admin / sub_admin 无限（-1）
- 体验额度用尽返 429 `quota_exceeded`：终身一次性，不按月重置——升级 ProMax 继续用（见 §9.8）；异常情况引导用户登录官网联系开发者

### 8.3 429 限流/配额

| 错误码 | 原因 | 兜底话术 |
|--------|------|----------|
| `rate_limited` | burst 30/min 触发 | 「请求太频繁啦，休息 1 分钟再来～」（Retry-After: 60） |
| `quota_exceeded` | 免费用户终身体验额度用完 / ProMax 本期配额用尽 | 免费用户：「您的 20 quota 免费体验额度已用完（终身一次性，不按月重置）。升级 ProMax（1000 quota / 30 天）可继续使用：登录 mrmodel.cesario.top → 会员中心；如遇异常欢迎登录官网联系我们处理」。ProMax：「本期 MCP 配额已用尽（X/1000），本期结束后自动重置；如遇异常欢迎登录官网联系我们处理」（Retry-After: 86400） |

### 8.4 5xx 服务端错误

| 错误码 | 原因 | 兜底话术 |
|--------|------|----------|
| 500 | MCP 端异常 | 「MCP 服务开小差了，请稍后重试～持续报错请登录官网联系我们处理」 |
| 503 | 服务临时不可用 | 「MCP 服务维护中，请稍后重试～给您带来不便敬请谅解」 |

---

## 9. FAQ

### 9.1 token 找不到 / 格式错误

**Q**：调 MCP 时报 `invalid_token`？
**A**：
1. 检查 `MR_MCP_TOKEN` 或 `~/.config/mrmodel/token` 是否设置
2. 检查 token 格式：`mcp_live_<48hex>`（共 57 字符）
3. token 注册即有（1 人 1 个），去 `https://mrmodel.cesario.top/mcp-tokens` 随时查看/复制完整明文；如怀疑泄露，在该页点「重置」，旧 token 立即失效
4. ⚠️ **不要**把 token 写到 git 仓 / 聊天历史 / 公开 issue

### 9.2 MCP 端不可达

**Q**：`curl https://mcp.cesario.top/healthz` 返非 200？
**A**：
1. 服务维护期间 — 等服务恢复
2. CF 代理问题：检查 `https://mcp.cesario.top` 是否能打开
3. 本地网络问题：检查 本机网络 / 热点

### 9.3 配额打爆

**Q**：报 `quota_exceeded`，配额怎么不见恢复？
**A**：
1. 免费体验额度 = 20 quota 终身一次性，不按月重置；用完升级 ProMax 即可继续
2. ProMax = 1000 quota / 30 天滚动窗口，本期用尽等本期结束自动重置（按开通时间起算，非自然月）
3. 用量随时在官网 mcp-tokens 页查看；如对扣费有疑问，欢迎登录官网联系开发者核对

### 9.4 行情 skill 未装

**Q**：希望拿到 PE/估值但没装 a-stock-data？
**A**：
```bash
# 从 GitHub 装（私仓，需要授权访问）
git clone https://github.com/simonlin1212/a-stock-data ~/.claude/skills/a-stock-data
```
装完后下次调用自动扫描识别。

### 9.5 触发词漏触发

**Q**：用户说"XX 怎么看"没说"模型先生"没触发？
**A**：
- 本 skill **不强制 100% 命中**（LLM 启发式），"模型先生"在 description 头部是强信号
- 用户口语化提问命中率 > 95%
- 漏触发时手动加「模型先生，XX 怎么看」即可

### 9.6 合规硬闸误判

**Q**：LLM 输出里出现"买入"等字眼？
**A**：
- LLM 必须在输出前自检（见 §4.6）
- 如出现：手动改写为"风控需关注..." / "看多逻辑在 X 条件下成立"等通用方法论话术
- 报告误判案例给开发者，下次版本更新 self-check prompt

### 9.7 跨设备同步

**Q**：在别的电脑/服务器也想用？
**A**：
```bash
# 在目标机器上
mkdir -p ~/.claude/skills/mr-model/
cp ~/.claude/skills/mr-model/SKILL.md ~/.claude/skills/mr-model/
cp ~/.claude/skills/mr-model/manifest.json ~/.claude/skills/mr-model/
# 配置 token（环境变量或 ~/.config/mrmodel/token）
```

### 9.8 如何升级到 ProMax

**Q**：20 quota 终身体验额度用完了，或报 `mcp_not_enabled`，怎么升级 ProMax？

**A**：

1. 登录 https://mrmodel.cesario.top → 头像 → 会员中心 → 选 ProMax
2. 支付开通：价格以官网首页公告为准
3. token 无需重新创建：注册即有，升级后同一 token 立即享 1000 quota / 30 天

**体验额度 vs ProMax**？

- 人人保底 20 quota 终身体验（所有账号状态正常的用户），轻量查询约可问 6 个问题
- ProMax 1000 quota / 30 天，重度分析不心疼；**6 高级 tool + 大配额 + 多用户共享 + 跨设备同步** = ProMax 核心价值
- 5 基础 tool 对应的查询能力在官网 Web 端免费（网页直接查视频/博主/评论）；MCP 通道的价值是让 AI 助手自动化调用全部 11 tool
- 体验额度用尽返 429 `quota_exceeded`：终身一次性不重置，升级 ProMax 立即恢复大配额

**升级后立即可调**：1 token 跨设备不区分（iPhone/Mac/Linux 同一 token 都享 1000 quota / 30 天，1 用户 1 API key 策略）

### 9.9 11 tool / 6 高级 tool 在哪看

**Q**：升级到 ProMax 后哪些高级 tool 立即可用？
**A**：6 高级 tool 全部立即可用，无额外开关：
- `query_real_desc_text` / `query_dimension_levels`（单视频深度解读）
- `query_transcript_keywords`（转录 5 类分析）
- `query_aggregated_sentiment`（多空情绪聚合 + 拐点）
- `query_creator_meta`（博主元信息）
- `query_trending_keywords`（平台热词趋势）

详见 §3.6 决策细化表 + §6.2 范本

---

## 附录 A：11 tool 输出结构参考（实测 2026-09-04，生产环境真机采样）

> **数据免责声明**：本平台数据来自第三方博主公开视频内容的采集聚合，可能存在采集延迟、字段缺失或博主主观表述偏差；本附录结构以 2026-09-04 生产实测为准，服务端升级后以 `tools/list` 实际返回为准。

### A.1 通用顶层字段（query_video_list / search_videos / query_blogger_opinions / search_video_transcripts 单条共有）

```json
{
  "aweme_id": "7681209106778645105",       // 抖音视频唯一 ID（**string 类型**，客户端存储时保留引号）
  "desc_text": "...",                      // 视频简介（占位符时会有 _desc_note）
  "create_time": 1788420861,               // epoch 秒
  "create_time_str": "2026-09-03 15:34",   // CST 字符串
  "duration": 85.5,                        // 视频时长（秒，float）
  "statistics": {"digg_count": 8868, "comment_count": 1458, "share_count": 921, "play_count": 0, "collect_count": 670},
  "tags": ["随拍", "生活记录", "日常vlog"],  // 平台采集标签
  "content_type": "video",
  "author_nickname": "模型先生",           // 博主名（当前唯一在网博主）
  "author_sec_uid": "MS4wLjABAAAA...",     // 抖音 sec_uid
  "_desc_note": "original_desc_is_placeholder_fallback_summary_used",  // 仅占位符时出现
  "dialectics_tags": ["综合"],             // 辩证维度标签（兜底['综合'] 8 维各 0.5）
  "framework_dimensions": {                // 8 维（实测全有, 兜底各 0.5）
    "估值类": {"score": 0.5, "description": "...", "suggested_data_sources": [...], "analysis_steps": [...]},
    "趋势类": {"score": 0.5, ...},
    "基本面类": {"score": 0.5, ...},
    "风险类": {"score": 0.5, ...},
    "逻辑类": {"score": 0.5, ...},
    "情绪类": {"score": 0.5, ...},
    "策略类": {"score": 0.5, ...},
    "择时类": {"score": 0.5, ...}
    // ⚠️ 注意：这 4 个 tool 里的单维只有 4 键（score/description/suggested_data_sources/analysis_steps），
    //    level + label 两键只在 query_dimension_levels 里才出现（见 A.3.2）
  },
  "analysis_framework": {                  // 第四层（v316）：辩证元框架 + 风险词 + 三时段 + 用法
    "dialectics_meta_prompt": "当你分析一条 A 股财经观点时...",
    "risk_vocabulary": ["情绪过热", "分化失序", "共识明牌化", "量价背离", "高位放量滞涨", "均线死叉确认", "支撑位失守", "业绩不及预期"],
    "time_horizon_template": {"short_term": {...}, "mid_term": {...}, "long_term": {...}},
    "usage_hint": "客户端 LLM 使用方法：1. 将 dialectics_meta_prompt 作为 system prompt 的一部分 ..."
  },
  "_meta": {"quota_cost": 1, "quota_remaining": 809},   // quota_cost 可信；quota_remaining 当前版本不可靠（见 §3.3 第 7 条）
  "_tx_id": "uuid4-xxxx"                                    // M3 注入追踪 ID
}
```

### A.2 特殊：query_comments 聚合统计视图（实测）

```json
{
  "aweme_id": "7681209106778645105",
  "total_comments": 1453,       // 评论总数（单视频上限 5000 条样本）
  "total_digg": 2084,           // 评论点赞总数
  "avg_digg": 1.43,             // 平均点赞（float）
  "max_digg": 410,              // 最高点赞
  "time_earliest": 1788420936,  // ⚠️ epoch 秒（非 ISO 字符串）
  "time_latest": 1788506176,    // ⚠️ epoch 秒（非 ISO 字符串）
  "top_keywords": [             // ⚠️ [词, 频次] 二元组数组（非字符串数组），jieba 中文分词 TOP 10
    ["先生", 278], ["捂脸", 196], ["科技", 124], ["流泪", 97], ["开学", 66],
    ["视频", 65], ["行情", 64], ["玫瑰", 60], ["大哥", 59], ["调整", 57]
  ],
  "_meta": {"quota_cost": 1},
  "_tx_id": "9d7805fa-..."
}
```

> top_keywords 含表情词/语气词属正常现象（UGC 词频中性统计），LLM 展示时自行过滤噪声词。

### A.3 高级 tool 输出结构（v1.1.0 新增）

#### A.3.1 query_real_desc_text

返回结构同 A.1（14 字段全量，dict 形态），但保证 `desc_text` 是原始完整 desc_text（不是占位符），`_desc_note` 标记透出。

#### A.3.2 query_dimension_levels（实测）

```json
{
  "aweme_id": "7677520767986234289",
  "dialectics_tags": ["综合"],
  "dimension_scores": {
    "估值类": {"score": 0.5, "description": "...", "suggested_data_sources": [...], "analysis_steps": [...], "level": 1, "label": "中性"},
    "趋势类": {"score": 0.8, "level": 2, "label": "强信号", ...},
    ... 8 维
    // ⚠️ 与 A.1 不同：本 tool 的单维多 level + label 两键（6 键）
  },
  "_meta": {...},
  "_tx_id": "..."
}
```

#### A.3.3 query_transcript_keywords（实测）

```json
{
  "aweme_id": "...",
  "word_freq": [{"word": "这个", "weight": 0.2942}, {"word": "车店", "weight": 0.2465}],  // ⚠️ 键是 weight（TF-IDF 权重）非 freq；Top50
  "entities": {
    "stock": [], "concept": [], "kol": [],        // v1 词典匹配未命中时为空数组
    "_ner_engine": "dict_match_v1",                // 引擎标识
    "_recall_warning": "v1 词典覆盖 30 主流股 + 200 概念 + 50 KOL..."  // ⚠️ 漏召回警告在 entities 内部（非顶级）
  },
  "pos_distribution": {"v": 39, "n": 25, "zg": 8, "x": 31, "m": 11},  // 词性标记→次数（jieba 词性符号）
  "key_sentences": ["他终于他把钱付完以后...", "..."],  // ⚠️ 纯字符串数组（非对象），Top5 按关键词命中排序
  "transcript_summary_prompt": "请用 200-500 字总结以下视频转录的关键论点，按 4 段式输出：...",  // 拼好给客户端 LLM 加工
  "_meta": {"quota_cost": 2},
  "_tx_id": "..."
}
```

#### A.3.4 query_aggregated_sentiment（实测）

```json
{
  "keyword": "光模块",
  "granularity": "weekly",
  "date_from": "2026-08-05",
  "date_to": "2026-09-04",
  "total_videos": 1,
  "neutral_count": 1,
  "weekly_distribution": {"2026-W32": {"neutral": 1, "videos": 1}},  // 桶键=ISO 周/月；桶内=分类计数+videos 总数
  "trend_inflection_points": [],   // 多空净差符号变化处（无拐点时空数组）
  "_meta": {"quota_cost": 2},
  "_tx_id": "..."
}
```

⚠️ **当前版本不返回** `bull_count` / `bear_count` / `bull_bear_ratio` / `top_bull_quotes` / `top_bear_quotes`（服务端词典未命中多空词时视频只计入 `videos`，多空计数随词典命中动态出现）；多空对比由客户端 LLM 自行归纳。0 命中时 `weekly_distribution: {}`（无 `_hint`，降级路径见 §3.5）。

#### A.3.5 query_creator_meta（实测）

```json
{
  "sec_uid": "MS4wLjABAAAA...",
  "author_nickname": "模型先生",
  "stats": {
    "total_videos": 503,
    "videos_last_30d": 21,
    "videos_last_7d": 4,
    "total_digg": 3183692,
    "total_comment": 460518,
    "total_share": 655867,
    "avg_duration_sec": 70.25,
    "max_gap_days": 71.08,
    "first_video_at": 1660542235,   // ⚠️ epoch 秒（非日期字符串）
    "last_video_at": 1788420861
  },
  "_meta": {"quota_cost": 1},
  "_tx_id": "..."
}
```

⚠️ 当前版本不透出 `activity_score` 字段（活跃度评分待权重定版后上线）。

#### A.3.6 query_trending_keywords（实测）

```json
{
  "window": {"days": 7, "from": "2026-08-28", "to": "2026-09-04"},
  "sort_by": "videos",
  "top_keywords": [   // ⚠️ dict 数组（非字符串数组），statistics 加权
    {"word": "光模块", "videos": 12, "total_digg": 5000, "total_comment": 800},
    ...
  ],
  "new_keywords": [   // ⚠️ dict 数组（非字符串数组），本窗口新出现
    {"word": "新词1", "videos": 2, "total_digg": 100},
    ...
  ],
  "rising_keywords": [   // ⚠️ 键名 growth_ratio（非 growth_rate），环比 > 1.5
    {"word": "CPO", "current_videos": 3, "prev_videos": 1, "growth_ratio": 3.0}
  ],
  "_meta": {"quota_cost": 2},
  "_tx_id": "..."
}
```

### A.4 0 命中格式（仅 query_blogger_opinions / search_videos 返 `_hint`）

```json
{
  "_hint": {
    "reason": "no_match",
    "tool": "query_blogger_opinions",
    "suggestion": "尝试简化关键词 / 扩时间窗口 / 检查拼写"
  },
  "_tx_id": "e78e05dc-d1d0-414f-..."
}
```

其余 tool 0 命中行为（实测 2026-09-04）：
- `query_aggregated_sentiment` → 空桶 `{"total_videos": 0, "weekly_distribution": {}}`，**无 _hint**（降级见 §3.5）
- `search_video_transcripts` → 空 content list（0 条 item）
- `query_comments` / `query_real_desc_text` / `query_dimension_levels` / `query_transcript_keywords` → 传不存在的 aweme_id 返 error dict（带 hint）
- `query_creator_meta` / `query_trending_keywords` → 恒有数据（不依赖关键词命中）

## 附录 B：变更日志

- **v1.3.3** (2026-09-04) — 生产实测校准（60 测试点全量 E2E 后修正）
  - 🔴 §2.3 连通性 curl 补 CF 必需头（User-Agent + Accept: text/event-stream，缺失实测 403/406 必失败）+ SSE `data:` 行解析说明 + 一行提取 tool 名命令
  - 🔴 §3.2 新增「JSON-RPC content 多 item 解析」教学：list[dict] 返回被 FastMCP 拆成 N 条 content item，客户端必须遍历 `result.content[]` 逐条 json.loads（最易踩坑）
  - 🔴 附录 A 全量真机重测（2026-09-04 生产采样）：comments 时间戳 epoch 秒 + top_keywords 二元组数组、tkw word_freq 键 weight + entities 内含 _ner_engine/_recall_warning、sentiment 实际字段（不返 bull/bear 计数与 quotes）、creator_meta 不透出 activity_score + 时间 epoch 秒、trending 三类词 dict 形态 + growth_ratio 键名、video 类 tool 单维 4 键 vs dimension_levels 单维 6 键（level/label）
  - 🟠 §3.2/§3.3/§3.1 参数默认值对齐 server schema（search_videos page_size=20 / query_blogger_opinions limit=20 / search_video_transcripts limit=20，配额示例数字连带修正）；删除 query_comments 幻影 `limit` 参数
  - 🟠 §3.5/§6.2.2 sentiment 0 命中行为修正：返空 weekly_distribution 无 _hint，补 LLM 降级路径（先放宽时间窗 → 降级 query_blogger_opinions 自行归纳）
  - 🟠 §3.3 第 7 条 quota_remaining 不可靠警告（实测恒定不递减，余量以官网 mcp-tokens 页为准）
  - 🟡 版本号三处对齐（frontmatter 1.3.3 / manifest 1.3.3 / README 徽章 1.3.3）；aweme_id 强调 string 带引号；§2.3 补传输层无状态说明（无 Mcp-Session-Id，initialize 可选）
  - 附录 A 头部补数据免责声明（第三方采集延迟/缺失/主观偏差提示）

- **v1.3.2** (2026-09-03) — 终身体验额度 + 话术升级
  - 配额语义治本：**20 quota = 终身体验额度（一次性赠送，不按月重置）**，修正「30 天窗口自动重置」旧表述；ProMax 1000 quota / 30 天滚动窗口不变（本期用尽等本期结束自动重置）
  - §8 全错误码兜底话术升级：语气客气安抚 + 异常情况引导登录官网联系开发者处理
  - §8.3 / §9.3 治本残留旧机制描述（「自然月 1 号归零」「下月窗口重置」）
  - §6.4 配额超限范本改双分支（免费终身型 / ProMax 窗口型）

- **v1.3.1** (2026-09-03) — 人人保底 20 体验
  - 配额档位改版：**所有账号状态正常的用户人人享 20 quota / 30 天体验**（废除 trial/plus/pro 锁死 0 旧语义）；ProMax 1000 / admin 无限不变
  - §8.2 错误码表移除 `mcp_not_available`（服务端已无此码）；403 话术对齐新配额语义
  - §9.8 升级指引改写（触发场景 = 体验额度用尽 429 或 `mcp_not_enabled`）
  - 全文内部词清理 + §6.4 配额重置机制描述对齐 30 天滚动窗口

- **v1.3.0** (2026-09-03) — token 注册即有
  - 1 人 1 token 免申请/免创建，完整明文随时在 mcp-tokens 页查看/复制（告别只展示一次），泄露点「重置」即换新（旧 token 立即失效）；错误兜底话术同步去「撤销/重新生成」流程

- **v1.2.1** (2026-09-02) — 文档去价格化
  - 价格数字全部移除（README + SKILL.md 话术），统一「以官网公告为准」——避免改价后装机文档过期撒谎
  - README 增「注册即享 20 quota 免费体验」引导；README/SKILL.md 增数据源说明（当前仅「模型先生」，后续接入更多博主以公告为准）

- **v1.1.0** (2026-08-27) — 5 痛点治本
  - **痛点 ①**：5 tool → 11 tool 决策树全表（新增 6 高级 tool：query_real_desc_text / query_dimension_levels / query_transcript_keywords / query_aggregated_sentiment / query_creator_meta / query_trending_keywords），frontmatter 同步更新
  - **痛点 ②**：新增"灵活模式"（v1.1.0 默认）+ "FULL 模式"双模式选择，§4.0 决策口诀，短问答/快查不再强制 11 字段块（省 token）
  - **痛点 ③**：§6.2 新增 6 高级 tool 范本（query_real_desc_text 14 字段 / query_aggregated_sentiment 拐点 / query_dimension_levels 8 维档位 / query_transcript_keywords 5 类分析 / query_creator_meta 博主 meta / query_trending_keywords 平台热词）+ §6.2.7 3 tool 组合范式
  - **痛点 ④**：§9.8 新增"如何升级到 ProMax" 3 步走 + ¥38.25 限时倒计时（8-27 剩 4 天，8-31 24:00 截止）；§8.2 错误码加 4 项业务策略要点
  - **痛点 ⑤**：§7.1 manifest URL 改 jsdelivr CDN 优先（max-age=604800），备选 raw 5min 边缘缓存坑说明；附录 A 扩 5→11 tool 输出结构
  - 附录 B 加 v1.1.0 变更日志

- **v1.0.0** (2026-08-26) — 初始版本
  - 5 tool 决策树 + 配额保护
  - 辩证法 FULL 模式（3 段式 + `<<<DIA>>>` 11 字段）
  - 合规硬闸 OP_RE 自检
  - 行情 skill 整合（白名单 + 启动扫描）
  - manifest 提示式自更新
  - 错误码全档映射
