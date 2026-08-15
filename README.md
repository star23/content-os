# ContentOS

**A self-improving daily intelligence briefing, run by a Claude Code agent.**

[中文文档 →](./README.zh-CN.md)

Every morning it reads ~90 curated X accounts, newsletters and podcasts across your
tracks, filters them against your topic library, and produces a briefing plus draft
posts — then grades yesterday's output and adjusts.

It is not a summarizer. The design premise is that **summarizing is cheap and judgment
is expensive**, so most of the system is about constraints: what counts as fresh, what
counts as duplicate, what you're not allowed to say.

---

## What it produces

| Output | Description |
|---|---|
| `daily_briefing_YYYY-MM-DD.md` | The briefing: market snapshot, ~20 must-reads, extended reading, topic candidates, drafts |
| `daily_data.json` | The same content as structured data, consumed by the dashboard |
| `daily_briefing.html` | A single-file dashboard — 7 tabs, no server, opens from disk |

See [`examples/sample-briefing-2026-08-06.md`](./examples/sample-briefing-2026-08-06.md)
for real output (drafts and personal analytics redacted).

---

## The three-phase loop

The system runs **learn → do → reflect**. The reflect phase is what makes it improve.

```
Phase 1  LEARN     Pull last 7 days of your own published posts + engagement.
                   Derive which hooks worked and which died. Update profile_recent.md.

Phase 2  DO        Scan all sources → filter → rank → write briefing + drafts,
                   weighted by what Phase 1 just learned.

Phase 3  REFLECT   Propose changes to the source registry and topic library
                   (promote / demote / add), plus positioning-drift observations.
                   Writes suggestions only — never edits your config directly.
```

Phase 3 proposing rather than applying is deliberate. An agent that silently rewrites
its own inputs will drift, and you won't see it happen.

---

## The constraints that do the real work

Most of the quality comes from a few hard rules, enforced in code rather than left to
the model's discretion:

**48-hour hard window.** Anything older than 48h is barred from must-reads regardless
of relevance. Violations raise an exception rather than degrade quietly. If there isn't
enough fresh material, a short briefing is the correct output — padding with stale items
is not.

**Event-level dedup.** Every item carries an `event_id`. One event gets one slot —
a different source or a different angle does not earn a second. Extra angles fold into
that item's `supporting[]` array. Without this, a big earnings day silently eats half
the briefing: in one real run, 20 items covered only 13 distinct events.

**Config is read-only to the agent.** `profile_longterm.md`, `info_sources.json` and
`topic_library.json` are never written by the agent. Suggestions land in
`daily_data.json` for you to accept by hand.

**Red lines beat performance data.** Your `profile_longterm.md` red lines are hard
constraints. A hook that performed well last week cannot override them.

---

## Setup

### Prerequisites

- [Claude Code](https://claude.com/claude-code)
- Two MCP servers — one to read the world, one to read yourself (details below)

### Data sources

The system needs two different reads on X, and they are not interchangeable:

| | What it reads | Used by |
|---|---|---|
| **[NewsLiquid](https://app.newsliquid.com?code=EycjLp)** | Other people's posts — search, user timelines, the ~90 accounts in `info_sources.json` | Phase 2 (the briefing itself) |
| **[Typefully](https://typefully.com/?via=star)** | Your own posts and their engagement metrics | Phase 1 (learning what worked) |

**NewsLiquid** supplies the raw material. Its MCP server exposes search plus per-user
timelines, which is what the source registry is built around. Worth knowing before you
tune your lists: per-account fetching is quota-limited, and when the quota trips, long
merged lists get truncated from the bottom — silently. That is the single most common
way this system misses a story. Keep lists short and purpose-named.

**Typefully** closes the loop. Phase 1 pulls your last 7 days of posts with real
engagement numbers, derives which hooks worked and which died, and writes that into
`profile_recent.md` — which then weights the drafts Phase 2 writes. Without it the
system still produces briefings, but it stops learning: you get a good reader and lose
the self-correction.

> Both links above are the author's referral links. Neither service is required by the
> code — swap in any MCP server that can fetch posts and your own analytics, and adjust
> `SKILL.md` Step 1 and Step 4 accordingly.

### 1. Clone and place your config

```bash
git clone https://github.com/<you>/content-os.git
cd content-os

cp config/info_sources.example.json   config/info_sources.json
cp config/topic_library.example.json  config/topic_library.json
cp config/profile_longterm.example.md profile_longterm.md
```

The `.gitignore` already excludes the un-suffixed copies — your real config and your
performance data stay local.

### 2. Fill in the three config files

**`profile_longterm.md`** — who you are, your tracks, your voice, your red lines.
This is the hard-constraint layer; the agent reads it and never writes it.

**`topic_library.json`** — the evergreen themes to match events against. The shipped
example is **schema only**; the tracks and performance notes have to be yours.

**`info_sources.json`** — the source registry. The shipped example is a real working
registry (~90 public accounts across AI / crypto / US equities / space / solo-business),
so it's usable as-is, but the value is in curating it for your own beat.

Source-list design matters more than it looks. Keep lists **small and purpose-named** —
when a rate limit hits, a 40-account merged list gets truncated and you lose whatever
sits at the bottom. That failure is silent. One real incident: a major model launch was
missed because the lab's account sat at position 31 of a 42-account list on a day the
API quota tripped.

### 3. Point the skill at your vault

`SKILL.md` refers to paths as `$VAULT_ROOT`. Set it to wherever your config and output
live, then invoke the skill from Claude Code.

### 4. Optional: run it on a schedule

`launchd` on macOS, `cron` on Linux. One gotcha worth knowing: a `launchd` job must have
`USER` set in its environment, or the Claude CLI loses all of its MCP connections and the
run degrades to a web-search-only briefing without saying so.

---

## Repository layout

```
SKILL.md                              The workflow — the actual system
config/
  info_sources.example.json           Source registry (real, working example)
  topic_library.example.json          Topic library (schema only)
  profile_longterm.example.md         Positioning + red lines (template)
templates/
  dashboard.html                      Single-file dashboard, data injected at build
examples/
  sample-briefing-2026-08-06.md       Real output, redacted
```

---

## What is not in this repo

The operator's own data, deliberately: engagement history, unpublished drafts,
the real topic library with its performance notes, and `profile_recent.md`.
The system is the constraints and the loop, not the data it accumulated.

---

## License

MIT — see [LICENSE](./LICENSE).
