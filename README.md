# M-Model — A 股财经视频 MCP 调用框架

[![Version](https://img.shields.io/badge/version-1.1.1-blue.svg)](https://github.com/Cesario121125/M-Model)
[![MCP Server](https://img.shields.io/badge/MCP-11_tools-green.svg)](https://mcp.cesario.top)
[![License](https://img.shields.io/badge/license-MIT-lightgrey.svg)](#license)

> 一行命令安装,11 个 MCP 工具,5 基础 + 6 高级,A 股财经视频全维度分析(博主观点/视频检索/转录/评论热词/多空情绪/8 维框架/平台热词)。

## 快速开始 (一行命令安装)

```bash
curl -sL https://cdn.jsdelivr.net/gh/Cesario121125/M-Model@main/scripts/install-mrmodel-skill.sh | bash
```

安装器会:
1. 写入 `~/.claude/skills/mr-model/SKILL.md` + `manifest.json`
2. 提示输入 MCP token(无则引导去 https://mrmodel.cesario.top/mcp-tokens 申请)
3. 探活验证(0 配额消耗)

**前置**:需要先注册 mrmodel 账号并在 https://mrmodel.cesario.top/mcp-tokens 创建 token。

## 它是什么

本仓库提供 **MCP 调用框架** + **输出规范化层** + **自更新外壳**,所有分析逻辑由 MCP 服务端 (`_DIALECTICS_META_PROMPT` v316 第四层) 注入,客户端 LLM 按 `<<<DIA>>>` 11 字段契约产出分析。

**它不是**:
- ❌ 投资框架 / 博主方法论 / 新分析体系
- ❌ 实时行情工具(联动 `a-stock-data` skill 获取 PE/K线/资金面)
- ❌ 个股直接买卖建议(合规硬闸硬挡)

## 仓结构

```
M-Model/
├── SKILL.md                          # skill 完整文档 (42KB, 11 tool 决策树 + 双模式模板)
├── manifest.json                     # 自更新元信息 (sha256 + URL + version)
├── scripts/
│   └── install-mrmodel-skill.sh      # 一行命令装机脚本 (内嵌 base64 SKILL.md)
└── README.md                         # 本文件
```

## 11 个 MCP 工具速查

### 5 基础工具 (成本 1/次)

| 工具 | 用途 | 必填参数 |
|------|------|----------|
| `query_video_list` | 视频列表(按时间倒序) | — |
| `search_videos` | 关键词搜索视频 | `query` |
| `query_blogger_opinions` | 博主对某主题观点时间线 | `keyword` (≥2 字) |
| `search_video_transcripts` | 转录片段关键词搜索 | `keyword` |
| `query_comments` | 视频评论聚合统计 | `aweme_id` |

### 6 高级工具

| 工具 | 用途 | 必填参数 | 成本 |
|------|------|----------|------|
| `query_real_desc_text` | 单视频 14 字段完整元信息 | `aweme_id` | 1/次 |
| `query_dimension_levels` | 8 维框架档位 (level 0/1/2) | `aweme_id` | 1/次 |
| `query_transcript_keywords` | 转录 5 类分析 (词频/NER/词性/关键句/摘要 prompt) | `aweme_id` | 2/次 |
| `query_aggregated_sentiment` | 多空情绪聚合 + 拐点 | `keyword` | 2/次 |
| `query_creator_meta` | 博主 meta (活跃度/影响力) | `sec_uid` (可选) | 1/次 |
| `query_trending_keywords` | 平台热词 (top/new/rising) | — | 2/次 |

详细决策树见 [SKILL.md §3](SKILL.md#3-11-tool-调用决策树)。

## 两种输出模式

### 灵活模式 (默认,~300-500 字)

适合:快查 / 短问答 / "简要说说" / 闲聊

3 段式结构:
1. 正反论据对照表
2. 转化条件清单
3. 分时段判断 (短期/中期/长期)

### 完整模式 (~800-1500 字)

适合:深度分析 / 持仓决策辅助 / 学术性研究 / 用户明确说"详细分析"

3 段式 + `<<<DIA>>>` 11 字段块 (主要矛盾/主要方面/看多/看空/多空比/量变质变/现象本质/必然偶然/博主视角/数据视角/交叉关系)

**判别口诀**:用户说"详细/完整/全面/深度/系统" → 完整模式;用户说"简要/简略/快速/简答/一句话" → 灵活模式;默认 → 灵活模式 (省 token)。

## 升级 ProMax

5 基础工具免费;6 高级工具 + 大配额仅 ProMax 会员可用(¥45/月,限时折扣见主仓)。升级步骤:

1. 登录 https://mrmodel.cesario.top
2. 头像 → 会员中心 → 选 ProMax
3. 支付开通 → 等 monitor.sh 异步同步(1-5 分钟)
4. 去 https://mrmodel.cesario.top/mcp-tokens 创建 MCP token
5. 写入 `~/.config/mrmodel/token`(权限 600)

ProMax 享 1000 次/月,1 token 跨设备通用(iPhone/Mac/Linux 同一 token)。

## 合规边界

**禁用字眼** (LLM 输出前自检):买入 / 卖出 / 加仓 / 减仓 / 止损 / 止盈 / 建仓 / 补仓 / 低吸 / 追入 / 右侧追 / 介入 / 目标价

**禁用内容**:具体仓位比例 / 具体点位 / 具体目标价止损价

**允许内容**:通用投资方法论 / 转化条件失效提示 / 数据事实陈述

末尾固定声明:"本内容属通用投资分析方法,不构成任何投资建议;用户自行决策并承担风险。"

## 链接

- **MCP 服务**: https://mcp.cesario.top (Bearer token 鉴权)
- **主仓 Web/API**: https://mrmodel.cesario.top
- **Token 管理**: https://mrmodel.cesario.top/mcp-tokens
- **完整文档**: [SKILL.md](SKILL.md)

## License

MIT — 自由使用,需保留版权声明。
