# llm-wiki-skill

A **Karpathy-style LLM Wiki** packaged as an [AgentSkill](https://github.com/openclaw/openclaw) for
[OpenClaw](https://openclaw.ai) / Claude Code / any skill-aware agent.

> Persistent, plain-markdown knowledge base where **the LLM writes and maintains all content**,
> and the user supplies sources, exploration direction, and questions.
> No vector database — an `index.md` plus Obsidian-style `[[wikilinks]]` is enough.

Inspired by Andrej Karpathy's "LLM wiki" pattern.

---

## ✨ Features

- **Three-layer structure**: `raw/` (sources) → `pages/` (LLM-maintained knowledge) → `index.md` (navigation)
- **Confidence tagging**: every page declares `EXTRACTED` / `INFERRED` / `AMBIGUOUS` / `UNVERIFIED` at both frontmatter and paragraph level — the LLM must surface this in answers
- **SHA256 dedup** for ingest sources (files / URLs), so you never re-index the same PDF twice
- **Two-layer lint**:
  - *Layer 1* — `lint.py` (mechanical scan: orphans, missing concept pages, stale pages, missing cross-refs, missing confidence tags, index consistency)
  - *Layer 2* — LLM walks the report and fixes issues interactively via MCP
- **Obsidian MCP first**: all reads/writes prefer the filesystem-sandboxed Obsidian MCP server, with `read`/`write`/`edit` fallback
- **Logged everything**: `log.md` is an append-only timeline of every ingest / query / lint operation

---

## 📦 Install

### As an OpenClaw skill

```bash
# Clone into OpenClaw's workspace skills dir
git clone https://github.com/<you>/llm-wiki-skill \
  ~/.openclaw/workspace/skills/llm-wiki
```

That's it — OpenClaw auto-discovers skills at startup.

### As a Claude Code skill

```bash
git clone https://github.com/<you>/llm-wiki-skill \
  ~/.claude/skills/llm-wiki
```

### As a generic agent skill

Copy the folder into whatever directory your agent scans for skills, or point the agent at
`SKILL.md` directly.

---

## ⚙️ Configuration

All paths are driven by environment variables with safe defaults:

| Variable | Default | Purpose |
|----------|---------|---------|
| `WIKI_ROOT` | `$HOME/.openclaw/workspace/wiki` | Where the markdown wiki lives |
| `WIKI_SKILL_DIR` | `$HOME/.openclaw/workspace/skills/llm-wiki` | Where this skill is installed (scripts referenced by SKILL.md) |
| `MCPORTER_CONFIG` | `$HOME/.openclaw/workspace/config/mcporter.json` | Optional — path to your [mcporter](https://github.com/CrazyPython/mcporter) config (for the Obsidian MCP server) |
| `FEISHU_TARGET` | *(empty)* | Optional — `user:ou_xxx` or `chat:oc_xxx` if you want `lint.py --notify` to push weekly reports via `openclaw message` |
| `FEISHU_CHANNEL` | `feishu` | Messaging channel name (forwarded to `openclaw message --channel`) |
| `AWS_REGION` / `WIKI_EMBED_SECRET` | `us-east-1` / `jarvis/opensearch` | Only used by the optional `embed.py` (see below) |

Set them once in your shell profile, agent env, or cron line:

```bash
export WIKI_ROOT=$HOME/my-wiki
export WIKI_SKILL_DIR=$HOME/.openclaw/workspace/skills/llm-wiki
```

---

## 🗂 Initial wiki layout

After install, create the empty skeleton (or let the first ingest create it):

```bash
mkdir -p "$WIKI_ROOT"/{raw,pages/{aws,ai,clients,projects,ops},.lint-history}
cat > "$WIKI_ROOT/index.md" <<'EOF'
# Wiki Index

_Pages auto-listed here by the LLM after each ingest._
EOF
touch "$WIKI_ROOT/log.md"
```

---

## 🔌 Dependencies

**Required:**
- Python ≥ 3.9 (stdlib only for `lint.py` / `dedup.py`)

**Recommended:**
- [mcporter](https://github.com/CrazyPython/mcporter) with an Obsidian filesystem MCP server pointing at `$WIKI_ROOT`.
  Example entry in `mcporter.json`:
  ```json
  {
    "servers": {
      "obsidian": {
        "type": "stdio",
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", "$WIKI_ROOT"]
      }
    }
  }
  ```
  Without mcporter the skill still works — the LLM falls back to ordinary `read` / `write` / `edit` tools.

**Optional / experimental:**
- `embed.py` — Bedrock Titan embeddings → OpenSearch indexing. Requires AWS creds, a secret named
  `$WIKI_EMBED_SECRET` containing `{endpoint, username, password}`, and `boto3 + opensearch-py +
  requests-aws4auth`. Skip unless you want semantic search on top of the wiki.

---

## 🚀 Usage

Once the skill is loaded, talk to your agent naturally:

| You say | Skill does |
|---------|-----------|
| "加进 wiki" / "ingest this" | Reads `index.md`, decides new-vs-update, writes page + updates index + logs |
| "查 wiki" / "what do we have on X" | Reads `index.md` + relevant pages, answers with `> 参考：[[page]]` citations |
| "整理 wiki" / "lint the wiki" | Runs `lint.py`, then LLM walks the report interactively to fix issues |

Under the hood the agent follows the flows in `scripts/ingest.md`, `scripts/query.md`, `scripts/lint.md`.

---

## 🗓 Weekly lint cron

```cron
# every Monday 02:00
0 2 * * 1 WIKI_ROOT=$HOME/.openclaw/workspace/wiki FEISHU_TARGET=user:ou_xxxxxxxxx \
  python3 $HOME/.openclaw/workspace/skills/llm-wiki/scripts/lint.py --notify \
  >> $HOME/.openclaw/workspace/wiki/.lint-history/cron.log 2>&1
```

If `FEISHU_TARGET` is unset, the lint still runs and writes a report — it just doesn't push.

---

## 🧭 Why this pattern?

Plain markdown + Obsidian-style links gives you:

- **Zero lock-in** — it's just `.md` files; any editor works
- **Version-control friendly** — your wiki content belongs in a separate (private) git repo
- **Grep-able forever** — no ORM, no schema migrations, no embeddings to rebuild
- **LLM-native** — every page fits in context, and the whole `index.md` is an agent's cognitive map

The LLM is responsible for *curation* (deduping, cross-referencing, contradiction detection),
not just bulk-dumping. Hence the confidence tags, the lint flow, and the append-only `log.md`.

---

## 📁 Repository layout

```
llm-wiki-skill/
├── SKILL.md              # skill manifest + operating rules (what the LLM reads)
├── scripts/
│   ├── ingest.md         # ingest flow spec
│   ├── query.md          # query flow spec
│   ├── lint.md           # lint flow spec (Layer 1 + Layer 2)
│   ├── lint.py           # Layer 1 mechanical scanner
│   ├── dedup.py          # SHA256 dedup cache for sources
│   └── embed.py          # optional: Bedrock Titan → OpenSearch
├── README.md             # you are here
└── LICENSE               # MIT
```

---

## 🔐 Separate your wiki content from this skill

**Do not commit your actual wiki (`$WIKI_ROOT`) to this public repo.**

This repo contains only the *skill definition*. Your wiki content — clients, account IDs,
decisions — should live in:

- a **separate private repo** (recommended), or
- a local Mutagen/rclone mount, or
- AWS S3 / any blob store

That separation is the whole point: the skill is reusable across machines; the knowledge is yours.

---

## 📝 License

MIT — see [LICENSE](./LICENSE).

## 🙏 Credits

Pattern inspired by [Andrej Karpathy](https://x.com/karpathy)'s "LLM wiki" idea.
Built for / battle-tested on [OpenClaw](https://openclaw.ai).
