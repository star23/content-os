# Star Daily Briefing

**一套会自我迭代的每日情报简报系统，由 Claude Code agent 驱动。**

[English →](./README.md)

每天早上扫描约 90 个精选 X 账号、newsletter 和播客，按你的选题库过滤，产出一份简报
和几条推文草稿——然后回头给昨天的产出打分，据此调整今天的权重。

它不是摘要工具。设计前提是：**摘要很便宜，判断很贵**。所以系统的大部分篇幅都在写约束——
什么算新、什么算重复、什么话不许说。

---

## 产出什么

| 产出 | 说明 |
|---|---|
| `daily_briefing_YYYY-MM-DD.md` | 简报正文：市场快照、约 20 条必看、延伸阅读、选题候选、草稿 |
| `daily_data.json` | 同样内容的结构化版本，供看板消费 |
| `daily_briefing.html` | 单文件看板，7 个 tab，无需服务器，本地双击打开 |

真实产出样例见 [`examples/sample-briefing-2026-08-06.md`](./examples/sample-briefing-2026-08-06.md)
（草稿和个人数据已脱敏）。

---

## 三阶段闭环

系统跑的是 **学 → 做 → 反**。让它持续变好的是"反"这一段。

```
Phase 1  学   拉取你近 7 天的发布记录和真实互动数据，
              推导哪些 Hook 生效、哪些失灵，写入 profile_recent.md。

Phase 2  做   扫描全部信息源 → 过滤 → 排序 → 写简报和草稿，
              权重由 Phase 1 刚学到的结论决定。

Phase 3  反   对信息源和选题库提出调整建议（提升 / 降级 / 新增），
              外加定位漂移观察。只写建议，绝不直接改你的配置。
```

Phase 3 只提建议不落地，是刻意的。一个会悄悄改写自己输入的 agent 一定会漂移，
而且你不会察觉。

---

## 真正起作用的是那几条硬约束

质量主要来自少数几条硬规则，而且是用代码强制、不交给模型自由裁量：

**48 小时硬窗口。** 超过 48h 的内容一律不准进必看，无论多相关。违规直接抛异常，
而不是悄悄降级。如果当天没有足够的新鲜素材，**简报短一点才是正确输出**，拿旧素材凑数不是。

**事件级去重。** 每条必看都带 `event_id`，同一事件只占 1 个名额——换个来源、换个角度
都不算例外，多出来的角度折进该条的 `supporting[]`。没有这条规则，一个大财报日会静悄悄
吃掉半份简报：某次真实运行里，20 条必看实际只覆盖了 13 个独立事件。

**配置对 agent 只读。** `profile_longterm.md`、`info_sources.json`、`topic_library.json`
永远不由 agent 写入。建议只落在 `daily_data.json` 里，由你手动采纳。

**红线压过表现数据。** `profile_longterm.md` 里的红线是硬约束，
上周表现再好的 Hook 也不能覆盖它。

---

## 安装使用

### 前置条件

- [Claude Code](https://claude.com/claude-code)
- 一个 X/Twitter 数据源（本仓库这套用的是暴露搜索和用户时间线的 MCP server）
- 可选：带分析数据的发布工具，供 Phase 1 使用。没有它系统会跳过学习阶段，简报照常产出。

### 1. 克隆并放置配置

```bash
git clone https://github.com/<你>/star-daily-briefing.git
cd star-daily-briefing

cp config/info_sources.example.json   config/info_sources.json
cp config/topic_library.example.json  config/topic_library.json
cp config/profile_longterm.example.md profile_longterm.md
```

`.gitignore` 已经排除了去掉 `.example` 后缀的那几份——你的真实配置和表现数据只留在本地。

### 2. 填写三份配置

**`profile_longterm.md`** — 你是谁、你的赛道、你的语气、你的红线。
这是硬约束层，agent 只读不写。

**`topic_library.json`** — 用来匹配事件的常青选题。仓库里这份**只有结构**，
赛道和表现备注必须换成你自己的。

**`info_sources.json`** — 信息源清单。仓库里这份是一份**真实可用**的清单
（约 90 个公开账号，覆盖 AI / 加密 / 美股 / 太空 / 一人公司），可以直接跑，
但真正的价值在于按你自己的领域重新筛。

信息源的**列表怎么分组比看上去重要**。原则是**列表要小、名字要说明用途**——
一旦撞上 API 限流，一个 40 人的合并列表会被截断，排在后面的账号就丢了，
而且这种失败是无声的。真实教训：某次配额触顶那天，一家实验室的官方号排在 42 人列表的
第 31 位，结果它的重要模型发布被整个漏掉。

### 3. 把 skill 指向你的目录

`SKILL.md` 里的路径都写成 `$VAULT_ROOT`，改成你放配置和产出的位置，
然后在 Claude Code 里调用这个 skill。

### 4. 可选：挂定时任务

macOS 用 `launchd`，Linux 用 `cron`。一个值得知道的坑：`launchd` 任务的环境里
**必须设 `USER`**，否则 Claude CLI 会丢掉全部 MCP 连接，整次运行降级成只有网页搜索的简报，
而且不会报错。

---

## 目录结构

```
SKILL.md                              工作流——系统本体
config/
  info_sources.example.json           信息源清单（真实可用示例）
  topic_library.example.json          选题库（仅结构）
  profile_longterm.example.md         定位与红线（模板）
templates/
  dashboard.html                      单文件看板，数据在构建时注入
examples/
  sample-briefing-2026-08-06.md       真实产出，已脱敏
```

---

## 这个仓库里没有什么

刻意不含运营者自己的数据：互动历史、未发布草稿、带表现备注的真实选题库，
以及 `profile_recent.md`。这套系统的价值是那些约束和那个闭环，不是它攒下来的数据。

---

## 许可

MIT，见 [LICENSE](./LICENSE)。
