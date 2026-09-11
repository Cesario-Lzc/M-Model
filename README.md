# M-Model — A 股财经视频 MCP 调用框架

[![Version](https://img.shields.io/badge/version-1.4.1-blue.svg)](https://github.com/Cesario-Lzc/M-Model)
[![MCP Server](https://img.shields.io/badge/MCP-17_tools-green.svg)](https://mcp.cesario.top)
[![License](https://img.shields.io/badge/license-MIT-lightgrey.svg)](#license)

> 17 个 MCP 工具，5 基础 + 6 高级 + 6 功能，A 股财经视频全维度分析（博主观点 / 视频检索 / 转录 / 评论热词 / 多空情绪 / 8 维框架 / 平台热词 / **结构化观点追踪 / 每日晨报 / 服务端关注清单 / 免费探额探测**）。
>
> **数据源**：目前仅收录财经博主「模型先生」的 A 股视频数据；后续将接入更多博主，以 [官网最新公告](https://mrmodel.cesario.top) 为准。

## 快速开始

```bash
# 这里刻意走 raw.githubusercontent（5 分钟缓存）而非 jsdelivr：
# 脚本里的 Step 4 负责写入版本请求头 X-Skill-Version，而 jsdelivr 有 7 天缓存，
# 命中旧版脚本会漏写该头 —— 那样即使重装，2026-09-22 起仍会被服务端拒绝。
curl -sL https://raw.githubusercontent.com/Cesario-Lzc/M-Model/main/scripts/install-mrmodel-skill.sh | bash
```

成品位置：装机完成后 `~/.claude/skills/mr-model/SKILL.md` + `manifest.json` 已就位，启动 Claude 时按 SKILL.md §1 触发词自动激活。

**前置**：注册 mrmodel 账号（[mrmodel.cesario.top](https://mrmodel.cesario.top)）→ 查看/复制 MCP token（[mcp-tokens](https://mrmodel.cesario.top/mcp-tokens)，**注册即有，1 人 1 个**）→ 写入 `~/.config/mrmodel/token`。

## 它能做什么

### 5 基础 tool（视频/搜索/评论）

| 能力 | 工具 | 输入 | 输出示例 |
|------|------|------|----------|
| 视频列表（支持增量） | `query_video_list` | `page=1` 或 `date_from/to` / `since_id` | 20 条 video 元信息（CST 日界过滤 / 增量游标） |
| 关键词搜视频 | `search_videos` | `query="中际旭创"` | 20 条命中视频 desc |
| 博主观点时间线 | `query_blogger_opinions` | `keyword="贵州茅台", limit=20` | 20 条视频的 8 维框架 + dialectics_tags |
| 转录片段搜 | `search_video_transcripts` | `keyword="光模块", limit=20` | 20 段转录 snippet（≤65 字含前后文） |
| 评论聚合（可选热评原文） | `query_comments` | `aweme_id="..."`, `include_samples=true` | 聚合统计 + TOP5 脱敏热评 |

### 6 高级 tool（单视频深挖/聚合/meta）

| 能力 | 工具 | 输出示例 |
|------|------|----------|
| 单视频全字段 | `query_real_desc_text` | 全字段元信息 + 8 维框架 |
| 8 维档位 | `query_dimension_levels` | 8 维 × {level 0/1/2 + label} |
| 转录 5 类分析 | `query_transcript_keywords` | 词频 Top50 + NER + 词性 + 关键句 + 摘要 prompt |
| 多空情绪聚合 | `query_aggregated_sentiment` | long/short 计数 + 比值 + 拐点 + TOP 引文 |
| 博主 meta | `query_creator_meta` | `{total_videos, videos_last_30d, ...}` |
| 平台热词 | `query_trending_keywords` | 50 词 × {top/new/rising} |

### 6 功能 tool（v1.4.0 新增，3 个免费）

| 能力 | 工具 | 成本 | 输出示例 |
|------|------|------|----------|
| 查配额余量 | `query_quota` | **免费** | `{limit, used, remaining, reset_at, is_lifetime}` |
| 探测新视频 | `check_new_video` | **免费** | `{latest_aweme_id, has_new}`（轮询神器） |
| 读关注清单 | `watchlist_get` | **免费** | `[{name, type}]`（服务端账户态，换设备不丢） |
| 写关注清单 | `watchlist_set` | 1 quota | 覆盖式 ≤50 条（stock/sector/concept/index） |
| 结构化观点追踪 | `query_stock_opinions` | 2+0.1/行 | **多标的批量**：空格分隔传个股/板块/概念，按标的分组返回历次看多/看空 + 时效 + 推理原文（"300308"也能查） |
| 每日晨报 | `get_daily_digest` | 8 quota | 当日新视频 + 关注清单多级命中（★★★/★★）+ 多空 + 评论热词 |

**调用事例**（每日自动化晨报，总 8 quota vs 散件 10+ quota）：

```python
# 每日定时：先免费探测，有更新才跑收费晨报
quota = call("query_quota")                                    # 免费
if call("check_new_video", known_id=last_id)["has_new"]:      # 免费
    digest = call("get_daily_digest", date="2026-09-07")      # 8 quota 一次拿齐
    # → 新视频 + watchlist 命中（direct★★★ 精确点名 / mentioned★★ 提及）+ 多空方向 + 评论热词
# 用户问"中际旭创最近怎么说"（结构化观点直达，cost=4）
opinions = call("query_stock_opinions", symbol_or_name="中际旭创", date_from="2026-08-08", limit=20)
# → direction / validity / reasoning / viewpoint_date 观点行，客户端 LLM 自行组织分析
```

## 配额成本

> **公式**：`cost = ⌈base + 行数 × per⌉ quota`（向上取整，防拖库；dict 返回走 base 单次）
> **单位**：**quota**（配额点；ProMax 享 1000 quota / 30 天滚动窗口，其余档位人人享 20 quota 终身体验额度，一次性不按月重置）

### 免费 tool（0 quota）

| Tool | 用途 |
|------|------|
| `query_quota` | 自动化流程开头探余额 |
| `check_new_video` | 高频轮询"更新了没" |
| `watchlist_get` | 读服务端关注清单 |

### 收费 tool

| Tool | base | per×N | 典型成本 |
|------|------|-------|----------|
| `query_video_list` / `search_videos` | 1 | 0.1×N | 20 行 → **3 quota** |
| `query_blogger_opinions` / `query_stock_opinions` | 2 | 0.1×N | 20 行 → **4 quota** |
| `search_video_transcripts` | 2 | 0.05×N | 20 段 → **3 quota** |
| `query_comments` / `query_real_desc_text` / `query_dimension_levels` / `query_creator_meta` | 1 | 0 | **1 quota** |
| `query_transcript_keywords` / `query_aggregated_sentiment` / `query_trending_keywords` | 2 | 0 | **2 quota** |
| `watchlist_set` | 1 | 0 | **1 quota** |
| `get_daily_digest` | 8 | 0 | **8 quota**（聚合轨，顶替散件 10+ quota 联调） |

## 免费体验与升级

**免费体验**：**所有账号均享 20 quota 终身体验额度**（注册即有，一次性赠送不按月重置，17 tool 全部可用，无需付费；轻量查询约可问 6 个问题）。

**升级 ProMax**：享 1000 quota / 30 天 + 17 tool 全量，价格以[官网会员页](https://mrmodel.cesario.top)公告为准。

**升级路径**：登录 [mrmodel.cesario.top](https://mrmodel.cesario.top) → 头像 → 会员中心 → 选 ProMax → 支付 → 约 1-5 分钟自动生效（token 注册即有，无需申请，升级后同一 token 直接享大配额）。

**1 token 跨设备通用**（iPhone / Mac / Linux 同一 token 都享 1000 quota / 30 天，1 用户 1 API key；完整明文随时在 [mcp-tokens 页](https://mrmodel.cesario.top/mcp-tokens)查看/复制，泄露点「重置」即换新；**watchlist 关注清单存服务端账户态，换设备不丢**）。

## 合规能力

本 skill 返回的是**博主观点、视频/转录/评论聚合、结构化 claims 等事实数据**，供您结合自有行情数据源和大模型进行分析。**不下发操作指令（买入/卖出/仓位等）**，**不替您作投资建议**。

## 保持更新

本 skill 建议保持最新版。重跑一行安装命令即可升级到最新版（脚本会顺带把你的客户端版本写入 MCP 配置）：

```bash
curl -fsSL https://raw.githubusercontent.com/Cesario-Lzc/M-Model/main/scripts/install-mrmodel-skill.sh | bash
```

首次运行时会询问是否授权「自动静默更新」，授权一次后后续版本会在检测到更新时自动覆盖、不再二次打扰。

## 链接

- **MCP 服务**：[mcp.cesario.top](https://mcp.cesario.top)（Bearer token 鉴权）
- **官网 Web / API**：[mrmodel.cesario.top](https://mrmodel.cesario.top)
- **Token 查看/复制/重置**：[mrmodel.cesario.top/mcp-tokens](https://mrmodel.cesario.top/mcp-tokens)（注册即有 · 1 人 1 个）
- **完整文档**：[SKILL.md](SKILL.md)（17 tool 决策树 + 双模式输出规范 + 功能 tool 范本 + 合规硬闸）

## License

MIT — 自由使用，需保留版权声明。
