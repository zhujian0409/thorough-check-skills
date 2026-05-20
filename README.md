# thorough-check

**Languages:** English | [中文](README.zh-CN.md)

A Codex- and [Claude Code](https://docs.claude.com/en/docs/agents-and-tools/claude-code/overview)-compatible skill by [@zhujian0409](https://github.com/zhujian0409) that exhaustively verifies a code/config change with command-output evidence, impact mapping, and default read-only reviewer agents — and refuses to conclude "OK" from vibes.

## What it does

The natural follow-up after making a change is "double-check this carefully." In practice that usually gets you a polite "looks good" — until the change breaks in prod an hour later.

`thorough-check` runs an evidence-first audit and will not sign off without showing concrete command output for each block:

| Block | What's verified |
|---|---|
| **0 — Impact map** | What changed, what it touches, and which files/contracts/jobs/cache/data states may be affected |
| **A — Change sites** | New strings present; old strings fully removed; no missed variants (e.g. `404/5xx` vs `404+5xx` vs `404 + 5xx`) |
| **B — Boundary conditions** | Threshold neighbors, null, empty, overflow-length — do extreme values still render correctly? |
| **C — File integrity** | Syntax legal; tags/brackets balanced; JSON/YAML/Python/HTML parses clean |
| **E — Call-chain context** | Upstream data source still matches; every downstream caller still compatible; full horizontal flow, state migration, rollback, old data, retry, queue, publish, and cache risks are accounted for |

For real verification runs, the skill defaults to read-only reviewer agents when the runtime supports them:

- an **impact reviewer** that independently searches for missed blast radius;
- a **final reviewer** that tries to disprove unsupported "passed" claims before the final answer.

Token cost is not a reason to skip those reviewers. If the runtime cannot spawn subagents, the final report must say so explicitly.

Each check produces a row in the final table with **Check / Result / Evidence**. Rows without a concrete grep/diff/parse output get marked ⚠️ and a demand for the actual command, not a "seems fine."

Output language auto-detects: English primary, Chinese when the conversation is primarily Chinese.

## When to use

This skill **auto-triggers** on phrases that express doubt about a just-made change, for example:

**English:**

- "Double-check everything carefully"
- "Are all the edge cases OK?"
- "Is the entire call chain consistent?"
- "Verify this change won't break anything"
- "Re-check thoroughly before you sign off"

**中文:**

- "改完了帮我再仔细检查一遍"
- "各种边界都 ok 吗"
- "流程链条都对吗"
- "能确定不会出问题吗"

Unlike `stakeholder-writeup`, this one is meant to fire automatically. Codex can use the skill description and `agents/openai.yaml` metadata, while Claude Code can pick up the trigger phrases because `disable-model-invocation` is intentionally absent.

## Installation

Clone this repo and copy the skill directory into your user-level skills folder.

For Codex:

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/zhujian0409/thorough-check-skills.git
cp -r thorough-check-skills/thorough-check ~/.codex/skills/
```

Or keep the repo and symlink it for Codex:

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/zhujian0409/thorough-check-skills.git
ln -s "$(pwd)/thorough-check-skills/thorough-check" ~/.codex/skills/thorough-check
```

For Claude Code:

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/zhujian0409/thorough-check-skills.git
cp -r thorough-check-skills/thorough-check ~/.claude/skills/
```

Or keep the repo and symlink it for Claude Code:

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/zhujian0409/thorough-check-skills.git
ln -s "$(pwd)/thorough-check-skills/thorough-check" ~/.claude/skills/thorough-check
```

Codex uses `~/.codex/skills/`; Claude Code uses `~/.claude/skills/`. Start a new session after installing or updating so the runtime picks up the latest skill metadata.

## Design notes

1. **Evidence, not vibes.** Every conclusion must be backed by a concrete command output. Better to show the user 20 verbose ✅/❌ rows than a single hand-waved "all OK."
2. **Walk the full chain, not just the changed line.** Most regressions come from the upstream data source or a downstream caller the change forgot about. The skill forces you to trace both directions.

See [SKILL.md](thorough-check/SKILL.md) for the complete checklist, default multi-agent review rules, typical "chain not fully checked" pitfalls, and the self-check the skill runs on itself after every audit.

See [CHANGELOG.md](CHANGELOG.md) for release history.

## License

MIT — see [LICENSE](LICENSE).
