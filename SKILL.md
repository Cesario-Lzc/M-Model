---
name: mr-model
description: 「模型先生」+ 任何问题（博主观点/视频检索/评论热词/最近 30 天对个股怎么看）→ 触发本 skill。内部按 5 tool 决策树调用 https://mcp.cesario.top（query_video_list / search_videos / query_blogger_opinions / search_video_transcripts / query_comments），用 mcp_tokens Bearer 鉴权，输出 3 段式分析（正反论据→转化条件→分时段判断）+ <<<DIA>>> 11 字段 FULL 块（主要矛盾/主要方面/看多/看空/多空比/量变质变/现象本质/必然偶然/博主视角/数据视角/交叉关系）+ 合规硬闸（禁个股买卖方向/仓位/价位）。自动联动 ~/.claude/skills/ 下已装行情数据 skill（a-stock-data 等）。需先设置 MR_MCP_TOKEN 环境变量或 ~/.config/mrmodel/token 文件。懒校验、不烧配额、manifest 提示式自更新。
origin: custom
version: 1.0.0
---

# mrmodel-skill — mr-model MCP 调用框架

> **本 skill 不是投资框架，不是博主方法论，不是新分析体系。**
> 它**只是**一个 MCP 调用框架 + 输出规范化层 + 自更新外壳。
> 所有分析逻辑由 MCP 端 `_DIALECTICS_META_PROMPT` (v316 第四层) 注入 + 客户端 LLM 按 `<<<DIA>>>` 契约产出。

## 1. 概述 + 触发词

### 何时激活
- 用户说 **「模型先生」+ 任何问题**（必含「模型先生」4 字触发，LLM 启发式识别）
- 常见别名（命中即触发）：「博主 30 天怎么看 XX」「找视频关于 XX」「MCP 鉴权测试」「查模型先生的视频」

### 适用场景
- ✅ A 股财经观点分析（博主视频/转录/评论）
- ✅ 个股/板块/题材的视频聚合检索
- ✅ 博主最近 X 天对某主题的观点时间线
- ✅ 行情数据（联动 a-stock-data skill）
- ❌ 非 A 股（美股/港股/期货）—— 走 mr-overseas-kline
- ❌ 个股直接买卖建议 —— 合规硬闸硬挡
- ❌ 主仓管理后台操作 —— 走主仓 admin 后台

### 核心约束
- **不动 A1**（A1 封锁铁律）— 本 skill 是用户本地工具，**不入任何 git 仓**
- **不动主仓**（B 档不动主仓铁律）— 只读 `mcp_tokens` 鉴权 + 调 5 tool
- **不入 vault**（vault 是写书素材库，技能工具集职责分离）

---

## 2. 前置（必读）

### 2.1 token 读取优先级（3 级 fallback）

```bash
# 优先级 1（推荐）：环境变量
export MR_MCP_TOKEN="mcp_live_<64hex>"

# 优先级 2（fallback）：文件，权限 600
mkdir -p ~/.config/mrmodel
echo -n "mcp_live_<64hex>" > ~/.config/mrmodel/token
chmod 600 ~/.config/mrmodel/token

# 优先级 3（都不存在）：报错兜底
# → 告诉用户去 https://mrmodel.cesario.top/mcp-tokens 生成
```

### 2.2 token 格式校验（启动时轻量预检，0 配额成本）

- 前缀：`mcp_live_`（**9 字符**：`m-c-p-_-l-i-v-e-_`）
- 后跟：**48 个十六进制字符**（`[0-9a-f]{48}`，192 位熵，`secrets.token_hex(24)` 生成）
- 总长：**57 字符**
- **格式正确 → 静默通过，第一次实际调用时才连 MCP 鉴权**
- **格式错误 → 立即报错 "token 格式不合法，应为 mcp_live_<48hex> 共 57 字符"**

### 2.3 测试连通性命令（可选）

```bash
# 5 tool 鉴权测试
curl -X POST https://mcp.cesario.top/mcp \
  -H "Authorization: Bearer $MR_MCP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"query_video_list","arguments":{"page":1,"page_size":1}}}'

# 期望：返 {"jsonrpc":"2.0","id":1,"result":{...}}，含 _tx_id 字段
```

---

## 3. 5 tool 调用决策树

### 3.1 决策树（用户问句 → 调哪个 tool）

```
用户问题
├─ 包含"最近"/"最新"/"今天"/"昨天" + "视频"？
│   └─ YES → query_video_list(page=1, page_size=20)  ← 默认 20 条
├─ 包含"博主最近"/"30 天"/"X 天" + "怎么看" + 个股/板块名？
│   └─ YES → query_blogger_opinions(keyword=..., date_from=-30d, date_to=今天, limit=10)
├─ 包含"转录里"/"说过"/"提过" + 关键词？
│   └─ YES → search_video_transcripts(keyword=..., limit=5)
├─ 包含"评论"/"评论区"/"热不热"？
│   └─ YES → 先 query_video_list 拿最新视频 → 读 dict.aweme_id → query_comments(aweme_id=...)
│            （query_comments 返的是聚合统计视图：total_comments/avg_digg/top_keywords，不是评论 list）
└─ 包含具体关键词（个股名/板块名/概念名）但无时间限定？
    └─ YES → search_videos(query=..., page=1, page_size=10)  ← 模糊兜底
```

### 3.2 5 tool 输入参数速查

| Tool | 必填 | 关键可选 | 默认值 | 配额成本 | 返回类型 |
|------|------|----------|--------|----------|----------|
| `query_video_list` | — | `page`, `page_size` | page=1, page_size=20 | 1/次 | **dict**（单条 video 全字段） |
| `search_videos` | `query` | `page`, `page_size` | page=1, page_size=10 | 1/次 | **dict**（page_size=1 时单条，>1 时 list 包 dict） |
| `query_blogger_opinions` | `keyword` (≥2字) | `date_from`, `date_to`, `limit` (1-20) | limit=10 | 1/次 | **dict**（0 命中时 `_hint.reason=no_match`） |
| `search_video_transcripts` | `keyword` | `limit` (1-20) | limit=5 | 1/次 | **dict**（含 transcript 30 字 snippet 列表） |
| `query_comments` | `aweme_id` | `limit` | limit=20 | 1/次 | **dict 聚合统计**（total_comments/avg_digg/max_digg/top_keywords，不是评论 list） |

**关键差异（实测 2026-08-26）**：
- ❌ 不是「list 包 dict」形态
- ✅ 全部 dict 形态（page_size=1 单条 / >1 时 list 包 dict）
- ✅ 顶级字段直接是 video 数据 + `_meta`（quota_cost/remaining） + `_tx_id`（uuid4）
- ✅ 0 命中时 query_blogger_opinions / search_videos 返 `_hint` dict 替代空结果

### 3.3 配额保护策略（KISS：宁可少调不烧配额）

1. **默认 page_size=20**（query_video_list / search_videos），单次 cost=1；超 20 提示用户"是否需要翻第 2 页"（page=2 需用户显式确认）
2. **query_blogger_opinions limit=10**（默认足够，覆盖博主典型 7-30 天观点）
3. **search_video_transcripts limit=5**（snippet 30 字 × 5 = 150 字，token 经济）
4. **query_comments 单次 1 个 aweme_id**（聚合统计，1 视频 = cost=1）
5. **不级联调用**：拿不到结果就告诉用户，不无限重试

### 3.4 决策树禁忌

- **不调 query_video_list 两次**（避免浪费）— 第一次拿不到想要的关键词结果时，转 search_videos
- **不重复调 query_blogger_opinions** 同一 keyword 同一时间窗（让 LLM 缓存前次结果）
- **不调 query_comments 整列表**（只对最新 1-2 个视频取评论，节省配额）

### 3.5 0 命中处理（`_hint` 字段识别）

MCP 端 v326 M3 治本：0 命中时返 `_hint` 字典替代 list 包空 dict：

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

---

## 4. 辩证法分析模板（FULL 模式）

> **复用 MCP 端 v316 第四层**：`_DIALECTICS_META_PROMPT` (mcp_server.py:257-267) + `_RISK_VOCABULARY` (mcp_server.py:270-279) + `_TIME_HORIZON_TEMPLATE` (mcp_server.py:282-298)
>
> **本节是 system prompt 拼装范本**，LLM 收到 MCP 透出的 `analysis_framework` 字段后，按以下结构组织输出。

### 4.1 system prompt 拼装（4 件套）

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
合规：本分析属通用投资方法论，不构成任何投资建议
```

### 4.2 输出结构（3 段式 + `<<<DIA>>>` 11 字段 FULL 块）

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

### 4.6 合规硬闸（必须落实，LLM 输出前自检）

❌ **禁用字眼**（命中 → 改写为通用方法论提示）：
- 买入 / 卖出 / 加仓 / 减仓 / 止损 / 止盈
- 建仓 / 补仓 / 低吸 / 追入 / 右侧追 / 介入 / 目标价

❌ **禁用内容**：
- 具体仓位比例（如"建议 30% 仓位"）
- 具体点位（如"在 45.20 元买入"）
- 具体目标价/止损价

✅ **允许内容**：
- 通用投资方法论（"风控需关注 X 条件"）
- 转化条件失效提示（"看多逻辑失效时需重新评估"）
- 数据/事实陈述（"当前 PE 分位 80%"）

**LLM 输出后自检清单**（失败 → 改写）：
1. 全文扫描禁用字眼 OP_RE
2. 检查是否含具体数字仓位/点位
3. 末尾固定加"本内容属通用投资分析方法，不构成任何投资建议；用户自行决策并承担风险"

### 4.7 主仓对齐要点（不可破坏）

参考主仓 `config/system_prompt.yaml v1.0.4`：
- **隐私红线 P0**：不透露 GLM/Claude 等模型名（用"技术实现细节不便透露"）
- **禁提数据源品牌**：涨乐/华泰/腾讯/东方财富/同花顺/任何 skill/API 统一用"本平台数据"
- **"凡指代博主都改博主"**（博主表达方式铁律）
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
    trigger_keywords: [PE, 估值, K线, 行情, 研报, 龙虎榜, 北向, 资金流, 财务, 公告, 换手率, 市值]
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
MCP 5 tool 不提供实时行情数据。
- 视频/博主/转录/评论维度 → 调 MCP 5 tool
- 行情/估值/财务/资金面 → 联动 a-stock-data skill
两者互补不重叠，组合使用得到完整分析。
```

### 5.4 行情 skill 不存在的兜底

若用户没装 a-stock-data：
- 仍返回博主观点 + 框架维度（不报错）
- 在输出末尾加一行"（未装行情 skill，建议安装 a-stock-data 获取实时 PE/K线/资金面数据）"

---

## 6. 输出范本

### 6.1 成功范本（典型用法）

**用户**：「模型先生，30 天对中际旭创怎么看？」

**LLM 行为**：
1. 触发 skill 加载
2. 匹配决策树 → `query_blogger_opinions(keyword="中际旭创", date_from=-30d, date_to=今天, limit=10)`
3. 拼 4 件套 system prompt（含 `_DIALECTICS_META_PROMPT` + 8 维 + 风险词 + 合规硬闸）
4. 输出 3 段式 + `<<<DIA>>>` 11 字段 FULL 块

**实测返回结构示例**（query_blogger_opinions 0 命中 / 命中 / 5 tool 通式，2026-08-26）：

```json
// 0 命中（query_blogger_opinions / search_videos 都会触发）
{"_hint": {"reason": "no_match", "tool": "query_blogger_opinions", "suggestion": "尝试简化关键词 / 扩时间窗口 / 检查拼写"}, "_tx_id": "e78e05dc-..."}

// 命中（dict 形态，page_size=1）
{
  "aweme_id": "7677520767986234289",
  "desc_text": " · · ",
  "create_time_str": "2026-08-24 17:01",
  "statistics": {"digg_count": 7131, "comment_count": 1315, "share_count": 612, "play_count": 0, "collect_count": 793},
  "tags": ["综合"],
  "author_nickname": "模型先生",
  "_desc_note": "original_desc_is_placeholder_fallback_summary_used",
  "dialectics_tags": ["综合"],
  "framework_dimensions": {"估值类": {"score": 0.5, "description": "...", "suggested_data_sources": [...], "analysis_steps": [...]}, "趋势类": {...}, "基本面类": {...}, "风险类": {...}, "逻辑类": {...}, "情绪类": {...}, "策略类": {...}, "择时类": {...}},
  "analysis_framework": {"dialectics_meta_prompt": "当你分析一条 A 股财经观点时...", "risk_vocabulary": ["情绪过热", "分化失序", ...], "time_horizon_template": {"short_term": {...}, "mid_term": {...}, "long_term": {...}}, "usage_hint": "客户端 LLM 使用方法..."},
  "_meta": {"quota_cost": 1, "quota_remaining": 999},
  "_tx_id": "c3f96c2c-938e-41f1-83q1-910571603678"
}

// query_comments 聚合统计视图
{"aweme_id": "7677520767986234289", "total_comments": 1315, "total_digg": 8523, "avg_digg": 6.5, "max_digg": 487, "time_earliest": "2026-08-24T17:30", "time_latest": "2026-08-25T18:45", "top_keywords": ["光模块", "国产替代", "..."], "_meta": {...}, "_tx_id": "cb13a820-..."}
```

### 6.2 0 命中范本

**用户**：「模型先生，30 天对 XX（无此标的视频）怎么看？」

**MCP 返**：
```json
{"_hint": {"reason": "no_match", "tool": "query_blogger_opinions", "suggestion": "尝试简化关键词 / 扩时间窗口 / 检查拼写"}}
```

**LLM 输出**：
> 未找到 30 天内关于"XX"的视频。建议：① 简化关键词 ② 扩时间窗口到 90 天 ③ 检查拼写。
> tx_id: `uuid-xxx`（可在主仓 mcp_usage_logs 查）

### 6.3 配额超限范本

**MCP 返 429**：`quota_exceeded`，`Retry-After: 86400`

**LLM 输出**：
> 本月 MCP 配额已用尽（X/Y），请等下月窗口重置（1 天 0 时归零）。
> 如需立即继续，可在主仓升级配额或新增 token。

### 6.4 鉴权失败范本

**MCP 返 401**：`invalid_token`

**LLM 输出**：
> token 无效或已吊销，请检查：
> 1. 确认 `MR_MCP_TOKEN` 环境变量或 `~/.config/mrmodel/token` 文件已正确配置
> 2. 确认 token 格式为 `mcp_live_<64hex>`
> 3. 重新生成：去 https://mrmodel.cesario.top/mcp-tokens → 撤销旧 token → 创建新 token

---

## 7. 自更新（manifest 提示式）

### 7.1 manifest.json 字段

```json
{
  "name": "mrmodel-skill",
  "version": "1.0.0",
  "min_mcp_server_version": "326",
  "skill_md_sha256": "<sha256-of-this-file>",
  "skill_md_url": "https://mrmodel.cesario.top/static/skills/mrmodel-skill/SKILL.md",
  "manifest_url": "https://mrmodel.cesario.top/static/skill-manifests/mrmodel-skill.json",
  "updated_at": "2026-08-26T10:00:00Z",
  "changelog": "v1.0.0 初始版本"
}
```

### 7.2 检查流程（首次调用时 lazy 触发）

LLM 第一次调 MCP tool 之前：

1. 读本地 `manifest.json` + `mtime`（文件最后修改时间）
2. **mtime < 1 小时 → 跳过检查**（避免短时间多次拉取）
3. 否则发 GET 到 `manifest_url`（**不**烧 MCP 配额，走主仓静态资源）
4. 对比 `skill_md_sha256`：
   - **一致** → 继续，**不**更新 mtime
   - **不一致** → **提示主人**："发现新版本 v1.x.x，changelog 摘要：...，是否更新？"
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
rm ~/.claude/skills/mrmodel-skill/manifest.json
# 下次调用时自动拉新
```

---

## 8. 错误码处理（全档映射）

### 8.1 401 未鉴权

| 错误码 | 原因 | 兜底话术 |
|--------|------|----------|
| `missing_bearer` | 漏了 Authorization 头 | 「请配置 MR_MCP_TOKEN 环境变量或 ~/.config/mrmodel/token 文件」 |
| `invalid_token` | token_hash 不匹配 / status≠active | 「token 无效或已吊销，请去主仓 mcp-tokens 重新生成」 |
| `expired` | expires_at < now | 「token 已过期，请去主仓 mcp-tokens 重新生成」 |

### 8.2 403 已鉴权但禁止

| 错误码 | 原因 | 兜底话术 |
|--------|------|----------|
| `account_disabled` | 账号已禁用 | 「账号已禁用，请联系主仓 admin」 |
| `account_banned` | 账号被封禁 | 「账号被封禁，请等待封禁结束或联系主仓 admin」 |
| `mcp_not_enabled` | 当前账号未开放 MCP | 「当前账号未开通 MCP 会员，请去主仓开通（¥45/月，限时 8.31 前 ¥38.25）」 |

### 8.3 429 限流/配额

| 错误码 | 原因 | 兜底话术 |
|--------|------|----------|
| `rate_limited` | burst 30/min 触发 | 「请求过于频繁，1 分钟后重试」（Retry-After: 60） |
| `quota_exceeded` | 本月 quota 用完 | 「本月 MCP 配额已用尽，请等下月窗口重置（1 天 0 时归零）」（Retry-After: 86400） |

### 8.4 5xx 服务端错误

| 错误码 | 原因 | 兜底话术 |
|--------|------|----------|
| 500 | MCP 端异常 | 「MCP 端异常，请稍后重试或联系主仓 admin」 |
| 503 | 服务临时不可用 | 「MCP 服务维护中，请稍后重试」 |

---

## 9. FAQ

### 9.1 token 找不到 / 格式错误

**Q**：调 MCP 时报 `invalid_token`？
**A**：
1. 检查 `MR_MCP_TOKEN` 或 `~/.config/mrmodel/token` 是否设置
2. 检查 token 格式：`mcp_live_<64hex>`（共 74 字符）
3. 去主仓 `https://mrmodel.cesario.top/mcp-tokens` 撤销旧 token，创建新的
4. ⚠️ **不要**把 token 写到 git 仓 / 聊天历史 / 公开 issue

### 9.2 MCP 端不可达

**Q**：`curl https://mcp.cesario.top/healthz` 返非 200？
**A**：
1. 主人 A1 封锁铁律生效期间（无主命不动 A1）— 等主人发话
2. CF 代理问题：检查 `https://mcp.cesario.top` 是否能打开
3. 本地网络问题：检查 Mac 是否在 Tailscale / 热点

### 9.3 配额打爆

**Q**：报 `quota_exceeded` 但还没到月底？
**A**：
1. 检查本月初是否有滥用（高频自动级联调用）
2. 配额窗口 = 自然月（1 号 0 时归零），不等月底
3. 在主仓 mcp-tokens 页查看用量

### 9.4 行情 skill 未装

**Q**：希望拿到 PE/估值但没装 a-stock-data？
**A**：
```bash
# 从 GitHub 装（私仓，需要主人授权访问）
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
- 报告误判案例给老王，下次版本更新 self-check prompt

### 9.7 跨设备同步

**Q**：在 NAS/枣园/A1 服务器也想用？
**A**：
```bash
# 在目标机器上
mkdir -p ~/.claude/skills/mrmodel-skill/
cp ~/.claude/skills/mrmodel-skill/SKILL.md ~/.claude/skills/mrmodel-skill/
cp ~/.claude/skills/mrmodel-skill/manifest.json ~/.claude/skills/mrmodel-skill/
# 配置 token（环境变量或 ~/.config/mrmodel/token）
```

---

## 附录 A：5 tool 输出结构参考（实测 2026-08-26）

### A.1 通用顶层字段（5 tool 共有）

```json
{
  "aweme_id": "v_id_xxx",                 // 抖音视频唯一 ID（query_comments 时是入参回显）
  "desc_text": "...",                      // 视频简介（占位符时会有 _desc_note）
  "create_time": 1787562102,               // epoch 秒
  "create_time_str": "2026-08-24 17:01",   // CST 字符串
  "duration": 47.4,                        // 视频时长（秒）
  "statistics": {"digg_count": 7131, "comment_count": 1315, "share_count": 612, "play_count": 0, "collect_count": 793},
  "tags": ["综合"],                        // 主仓采集标签（空时 L1 词典兜底打）
  "content_type": "video",
  "author_nickname": "模型先生",           // 博主名（v316 update 唯一在网博主）
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
  },
  "analysis_framework": {                  // 第四层（v316）：辩证元框架 + 风险词 + 三时段 + 用法
    "dialectics_meta_prompt": "当你分析一条 A 股财经观点时...",
    "risk_vocabulary": ["情绪过热", "分化失序", "共识明牌化", "量价背离", "高位放量滞涨", "均线死叉确认", "支撑位失守", "业绩不及预期"],
    "time_horizon_template": {"short_term": {...}, "mid_term": {...}, "long_term": {...}},
    "usage_hint": "客户端 LLM 使用方法：1. 将 dialectics_meta_prompt 作为 system prompt 的一部分 ..."
  },
  "_meta": {"quota_cost": 1, "quota_remaining": 999},   // 配额扣费信息
  "_tx_id": "uuid4-xxxx"                                    // M3 注入追踪 ID
}
```

### A.2 特殊：query_comments 聚合统计视图

```json
{
  "aweme_id": "7677520767986234289",
  "total_comments": 1315,       // 评论总数
  "total_digg": 8523,           // 评论点赞总数
  "avg_digg": 6.5,              // 平均点赞
  "max_digg": 487,              // 最高点赞
  "time_earliest": "2026-08-24T17:30",   // 最早评论时间
  "time_latest": "2026-08-25T18:45",     // 最晚评论时间
  "top_keywords": ["光模块", "国产替代", "..."],   // 关键词 TOP 10
  "_meta": {"quota_cost": 1, "quota_remaining": 998},
  "_tx_id": "cb13a820-..."
}
```

### A.3 0 命中格式

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

## 附录 B：变更日志

- **v1.0.0** (2026-08-26) — 初始版本
  - 5 tool 决策树 + 配额保护
  - 辩证法 FULL 模式（3 段式 + `<<<DIA>>>` 11 字段）
  - 合规硬闸 OP_RE 自检
  - 行情 skill 整合（白名单 + 启动扫描）
  - manifest 提示式自更新
  - 错误码全档映射
