# mr-model MCP · 15 tool 返回结构参考（OUTPUT-REFERENCE v1.5.8）

> **本文是 SKILL.md 的附属参考文件**（随 install 脚本一起装入 skill 目录），完整 JSON 字段结构在这里，SKILL.md 正文只留指针——按需 Read 本文件，省 token。
> 数据免责声明：数据来自第三方博主公开视频的采集聚合，可能存在采集延迟、字段缺失或博主主观偏差；结构以生产实测为准，服务端升级后以 `tools/list` 实际返回为准。

## 返回结构总览（A.1-A.4）

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
  "_meta": {"quota_cost": 1, "quota_remaining": 809, "data_as_of": "2026-09-15 10:52"},   // quota_cost 可信；quota_remaining 读只读副本有分钟级延迟（见 §3.3 第 9 条）；data_as_of = 数据集最新视频时间（新鲜度外显，全 tool 通用）
  "_tx_id": "uuid4-xxxx"                                    // M3 注入追踪 ID
}
```

> ⚠️ **v1.4.0 起 `analysis_framework` 字段已全线下线**（辩证元框架 prompt/风险词表/三时段模板不再随视频返回），
> 客户端 LLM 基于 `dialectics_tags` + `framework_dimensions` + 事实数据自行组织分析（见 §4.1）。

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
  "top_keywords": [             // ⚠️ [词, 频次] 二元组数组（非字符串数组），jieba 中文分词 TOP 10（已滤称呼/表情/时间/平台通用等噪声词，v1.5.8）
    ["有色", 188], ["加息", 84], ["周期", 64], ["科技", 61], ["小票", 55],
    ["市场", 50], ["大盘", 42], ["行情", 38], ["主力", 30], ["调整", 27]
  ],
  "_meta": {"quota_cost": 1},
  "_tx_id": "9d7805fa-..."
}
```

> top_keywords 返回有信息量的实词（财经词/题材词），称呼与表情类噪声词已在服务端过滤。

**可选：`include_samples=true` 返 TOP5 评论原文（v1.4.0 新增，同 1 quota 不额外收费）**：

```json
{
  "samples": [
    {"text": "液冷板块，服务器产量爬坡...", "digg_count": 2681, "time_ts": 1788506176}
  ]
}
```

> 隐私边界：不含评论者昵称/uid 等任何标识，不含手机号/邮箱/身份证等个人信息，最多 5 条。
> 默认 `include_samples=false` 不返原文（合规默认行为不变）。

### A.3 高级 tool 输出结构（v1.1.0 新增）

#### A.3.1 query_real_desc_text

返回结构同 A.1（全字段 dict 形态），但保证 `desc_text` 是原始完整 desc_text（不是占位符），`_desc_note` 标记透出。

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

#### A.3.4 query_aggregated_sentiment（v1.4.0 字段更新）

```json
{
  "keyword": "光模块",
  "granularity": "weekly",
  "date_from": "2026-08-05",
  "date_to": "2026-09-04",
  "total_videos": 5,
  "long_count": 3,
  "short_count": 1,
  "neutral_count": 1,
  "long_short_ratio": 3.0,
  "weekly_distribution": {"2026-W35": {"long": 2, "short": 0, "neutral": 1, "videos": 3}},  // 桶键=ISO 周/月；桶内 long/short/neutral + videos 总数
  "top_long_quotes": ["snippet ≤30 字", ...],   // TOP 3 多头引文
  "top_short_quotes": ["..."],                  // TOP 3 空头引文
  "trend_inflection_points": [{"bucket": "2026-W36", "from": "long", "to": "short", "delta": -2, "net": -1}],
  "_meta": {"quota_cost": 2},
  "_tx_id": "..."
}
```

⚠️ 多空分桶依赖服务端行业多空词典命中，未命中视频只计入 `videos`；v1.4.0 起 `bull/bear` 字段名已改 `long/short`（旧名从未在响应中透出，无兼容包袱）。0 命中时 `weekly_distribution: {}`（无 `_hint`，降级路径见 §3.5）。

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

#### A.3.7 query_quota（v1.4.0 实测，0 quota 免费）

```json
{
  "quota_limit": 1000,                          // -1 = 不限（admin）
  "quota_used": 3,
  "quota_remaining": 997,                       // null = 不限
  "window_started_at": "2026-08-25T09:58:08+08:00",
  "reset_at": "2026-09-24T09:58:08+08:00",      // 窗口重置时间；终身体验额度为 null
  "is_lifetime": false,                         // true = 200 quota 终身体验额度
  "_meta": {"quota_cost": 0},
  "_tx_id": "..."
}
```

#### A.3.8 check_new_video（v1.4.0 实测，0 quota 免费）

```json
{
  "latest_aweme_id": "7682690967342409329",
  "latest_create_time": 1788820861,
  "latest_create_time_str": "2026-09-07 21:21",
  "has_new": true,                              // 传 known_id 比对；known_id 已删除时保守按 true
  "_meta": {"quota_cost": 0},
  "_tx_id": "..."
}
```

#### A.3.11 query_stock_opinions（v1.4.1 多标的批量，base 2 + 0.1/行）

**返回 dict**：key = 输入的每个实体（空格分隔），value = `{hit, claims}`，claims 内每行结构：

```json
{
  "中际旭创": {
    "hit": true,
    "claims": [
      {
        "claim_id": "a1b2c3d4e5f6",                  // 稳定锚点（aweme_id+entity+direction 派生），同观点重查同 id
        "aweme_id": "7661149563777223611",
        "entity_name": "中际旭创",                     // 标准化剥代码后缀
        "entity_name_raw": "中际旭创(300308)",         // 原始名（可能带代码）
        "entity_type": "stock",                       // stock/sector/concept/index/commodity
        "direction": "看多",                           // 看多/强烈看多/看空/强烈看空/中性/观察（观察=只是提及没给观点；博主观点客观陈述）
        "validity": "mid_term",                       // short_term/mid_term/long_term/event_driven
        "time_horizon_text": "半年内",                 // 自由文本，可空
        "timeliness": 0.85,                           // 0-1 时效分
        "quote": "704亿就是704亿，市场只信订单...",     // 博主原话金句（逐字摘自转录，≤50字，可能为空）
        "reasoning": "北美大客户 1.6T 招标提前...",     // 博主推理原文
        "viewpoint_date": "2026-09-05",               // 观点日期
        "video_summary": "本期讲光模块三剑客...",
        "create_time": 1786920861,
        "create_time_str": "2026-09-05 10:52"
      }
    ]
  },
  "光模块": {"hit": true, "claims": [ /* ... */ ]},
  "科创板": {"hit": false, "claims": []},
  // 每个命中实体另附 top_quotes：该标的博主观点评级最高的原话金句 ≤3 条
  //   （时间序去重，元素 {quote, viewpoint_date, aweme_id}，无金句则不带此字段）
  "_meta": {"quota_cost": 3},
  "_tx_id": "..."
}
```

> **dict 返回，FastMCP 只拆 1 条 content item**（区别于 list 返回的 N 条），解析见 §3.2。无命中实体返 `{"hit": false, "claims": []}`，全部 0 命中返 `{"_hint": {...}}`。名称匹配双向：「中际旭创」命中「中际旭创(300308)」；「300308」也能命中。

#### A.3.12 get_daily_digest（v1.5.6 实测，动态计费 1.5/期 ceil：0 期 0 / 1 期 2 / 近 5 期 8）

```json
{
  "date": "2026-09-21",                          // recent 模式=窗口内最新一期日期；day 模式=指定日
  "mode": "recent",                              // recent=近 5 期滚动（不传 date 默认）/ day=指定日
  "date_range": "2026-09-16 ~ 2026-09-21",       // 仅 recent 模式返回，窗口时间跨度
  "generated_at": "2026-09-22T13:55:09+08:00",
  "new_video_count": 5,
  "new_videos": [
    {
      "aweme_id": "...", "desc_text": "...", "create_time_str": "2026-09-21 17:07",
      "dialectics_tags": ["综合"], "framework_dimensions": {...},
      "direction": "中性",                        // 多空方向（看多/看空/中性，include_sentiment=true 默认附）
      "comment_top_keywords": [["先生", 106], ["今天", 71], ["发型", 36]],  // 每条视频 TOP3 评论热词
      "comment_analyzed": 272,                    // 库内有效评论条数（与平台快照 comment_count 口径不同）
      "quote": "博主本期原话金句"                  // 逐字转录摘取（≤100 字，无则不带此字段）
    }
  ],
  "_meta": {"quota_cost": 8},                    // 动态计费 = ⌈期数 × 1.5⌉，本例 5 期 → 8；0 期 → 0（不扣费）
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

v1.4.0 新增 tool 0 命中/边界行为：
- `query_stock_opinions` → 标的无观点返空 list（0 条 content item）；纯数字代码也能命中（双向匹配）
- `get_daily_digest` → **不传 date 恒返近 5 期**（当天没更新也满载，只有空库才返 `new_video_count: 0`）；传 date 该日无新视频返 `new_video_count: 0` + 空 `new_videos`（dict 恒有，不报错）；0 期返回 **0 quota 不扣费**（动态计费按期数）
- `check_new_video` → `known_id` 不存在（已删）保守按 `has_new: true`
