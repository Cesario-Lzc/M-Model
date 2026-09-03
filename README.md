# M-Model — A 股财经视频 MCP 调用框架

[![Version](https://img.shields.io/badge/version-1.3.1-blue.svg)](https://github.com/Cesario-Lzc/M-Model)
[![MCP Server](https://img.shields.io/badge/MCP-11_tools-green.svg)](https://mcp.cesario.top)
[![License](https://img.shields.io/badge/license-MIT-lightgrey.svg)](#license)

> 11 个 MCP 工具，5 基础 + 6 高级，A 股财经视频全维度分析（博主观点 / 视频检索 / 转录 / 评论热词 / 多空情绪 / 8 维框架 / 平台热词）。
>
> **数据源**：目前仅收录财经博主「模型先生」的 A 股视频数据；后续将接入更多博主，以 [官网最新公告](https://mrmodel.cesario.top) 为准。

## 快速开始

```bash
curl -sL https://cdn.jsdelivr.net/gh/Cesario-Lzc/M-Model@main/scripts/install-mrmodel-skill.sh | bash
```

成品位置：装机完成后 `~/.claude/skills/mr-model/SKILL.md` + `manifest.json` 已就位，启动 Claude 时按 SKILL.md §1 触发词自动激活。

**前置**：注册 mrmodel 账号（[mrmodel.cesario.top](https://mrmodel.cesario.top)）→ 查看/复制 MCP token（[mcp-tokens](https://mrmodel.cesario.top/mcp-tokens)，**注册即有，1 人 1 个**）→ 写入 `~/.config/mrmodel/token`。

## 它能做什么

| 能力 | 工具 | 输入 | 输出示例 |
|------|------|------|----------|
| 视频列表 | `query_video_list` | `page=1, page_size=20` | 20 条 video 元信息（按时间倒序） |
| 关键词搜视频 | `search_videos` | `query="中际旭创"` | 10 条命中视频 desc |
| 博主观点时间线 | `query_blogger_opinions` | `keyword="贵州茅台", limit=10` | 10 条视频的 8 维框架 + dialectics_tags |
| 转录片段搜 | `search_video_transcripts` | `keyword="光模块", limit=5` | 5 段 30 字 snippet（含前后文） |
| 评论聚合 | `query_comments` | `aweme_id=7412345678` | `{total, avg_digg, top_keywords: [...]}` |
| 单视频全字段 | `query_real_desc_text` | `aweme_id=7412345678` | 14 字段全量 + `analysis_framework` |
| 8 维档位 | `query_dimension_levels` | `aweme_id=7412345678` | 8 维 × {level 0/1/2 + label} |
| 转录 5 类分析 | `query_transcript_keywords` | `aweme_id=7412345678` | 词频 Top50 + NER + 词性 + 关键句 + 摘要 prompt |
| 多空情绪聚合 | `query_aggregated_sentiment` | `keyword="贵州茅台", granularity=weekly` | `{bull:8, bear:3, ratio:2.67, 拐点: [W23]}` |
| 博主 meta | `query_creator_meta` | `sec_uid` 可选 | `{total_videos: 850, activity_score: 0.92}` |
| 平台热词 | `query_trending_keywords` | `days=7, top_n=50` | 50 词 × {top/new/rising} |

**调用事例**（3 工具组合范式，成本 4 次 vs N 次联调）：

```python
# 用户问"贵州茅台最近 30 天怎么说"
# 1) 时间线聚合（cost=3）
opinions = call("query_blogger_opinions", keyword="贵州茅台", date_from="2026-07-28", limit=20)
# 2) 多空情绪 + 拐点（cost=2）
sentiment = call("query_aggregated_sentiment", keyword="贵州茅台", granularity="weekly")
# 3) 平台热词看大市风向（cost=2）
trending = call("query_trending_keywords", days=7, top_n=20)
# → 客户端 LLM 拼 3 段式 + 11 字段 DIA 块（详 SKILL.md §4.2）
```

## 配额成本

> **公式**：`cost = ⌈base + 行数 × per⌉ quota`（向上取整，防拖库；dict 返回走 base 单次）
> **单位**：**quota**（配额点，30 天滚动窗口；ProMax 享 1000 quota / 30 天，其余档位人人享 20 quota / 30 天体验）

### 5 基础 tool

| Tool | base | per×N | 默认值成本 |
|------|------|-------|-----------|
| `query_video_list` | 1 | 0.1×N | page_size=20 → **3 quota** |
| `search_videos` | 1 | 0.1×N | page_size=10 → **2 quota** |
| `query_blogger_opinions` | 2 | 0.1×N | limit=10 → **3 quota** |
| `search_video_transcripts` | 2 | 0.05×N | limit=5 → **3 quota** |
| `query_comments` | 1 | 0（dict 聚合） | 1 aweme_id → **1 quota** |

### 6 高级 tool（per_row=0，dict 返回走 base）

| Tool | base | 单次成本 |
|------|------|----------|
| `query_real_desc_text` | 1 | 1 quota |
| `query_dimension_levels` | 1 | 1 quota |
| `query_transcript_keywords` | 2 | 2 quota |
| `query_aggregated_sentiment` | 2 | 2 quota |
| `query_creator_meta` | 1 | 1 quota |
| `query_trending_keywords` | 2 | 2 quota |

## 免费体验与升级

**免费体验**：**所有账号均享 20 quota / 30 天免费体验**（注册即有，11 tool 全部可用，无需付费；轻量查询约可问 6 个问题）。

**升级 ProMax**：享 1000 quota / 30 天 + 11 tool 全量，价格以[官网会员页](https://mrmodel.cesario.top)公告为准。

**升级路径**：登录 [mrmodel.cesario.top](https://mrmodel.cesario.top) → 头像 → 会员中心 → 选 ProMax → 支付 → 约 1-5 分钟自动生效（token 注册即有，无需申请，升级后同一 token 直接享大配额）。

**1 token 跨设备通用**（iPhone / Mac / Linux 同一 token 都享 1000 quota / 30 天，1 用户 1 API key；完整明文随时在 [mcp-tokens 页](https://mrmodel.cesario.top/mcp-tokens)查看/复制，泄露点「重置」即换新）。

## 合规边界

**禁用**：买入 / 卖出 / 加仓 / 减仓 / 止损 / 目标价 / 仓位比例 / 具体点位（操作类）；稳赚 / 必涨 / 无风险 / 百分百 / 包赚 / 保底（收益承诺类，v1.2.0）；基于用户持仓/风险偏好的个性化建议。

**允许**：通用方法论 / 转化条件失效提示 / 数据事实陈述。详 SKILL.md §4.6 + §10。

## 链接

- **MCP 服务**：[mcp.cesario.top](https://mcp.cesario.top)（Bearer token 鉴权）
- **官网 Web / API**：[mrmodel.cesario.top](https://mrmodel.cesario.top)
- **Token 查看/复制/重置**：[mrmodel.cesario.top/mcp-tokens](https://mrmodel.cesario.top/mcp-tokens)（注册即有 · 1 人 1 个）
- **完整文档**：[SKILL.md](SKILL.md)（约 44KB，11 tool 决策树 + 双模式模板 + 6 高级 tool 范本 + 合规硬闸 + 合规声明常量）

## License

MIT — 自由使用，需保留版权声明。
