---
name: thorough-check
description: Exhaustively verify a code/config change you just made — covers change sites, boundary conditions, file integrity, and the upstream/downstream call chain around the modified code. Refuses to claim "OK" without concrete command output as evidence. Triggered by phrases like "double-check this carefully", "are all the edge cases OK?", "does the whole call chain still work?", "are you sure nothing's broken?" — or Chinese equivalents such as "改完了帮我再仔细检查一遍" / "各种边界都 ok 吗" / "流程链条都对吗" / "能确定不会出问题吗".
allowed-tools: Read Grep Bash
---

# Thorough Check skill (thorough-check)

## Core principle

**Evidence, not vibes.** Every conclusion must be backed by concrete command output. Better to list 20 verbose ✅/❌ items for the user than to hand-wave "all OK".

### Typical user trigger phrases

**English:**

- "Double-check everything carefully"
- "Are all the edge cases OK?"
- "Is the entire call chain consistent?"
- "Verify this change won't break anything"
- "Re-check thoroughly before you sign off"

**中文 (Chinese):**

- "仔细反复的检查 确定都没问题是吧"
- "各种边界 各种情况 各种修改地方的流程链条 都是 ok 的是吧"
- "多次分析 仔细检查"
- "改完仔细检查下"
- "确定没问题再放过"

What the user wants is for **every implicit worry in your head to be made explicit and verified** — not a polite "all good".

## Language policy

This skill's **final report** (the check table shown to the user) defaults to **English** and auto-switches to **Chinese** when:

- The user's last 3 messages are primarily Chinese, OR
- The session history is primarily Chinese, OR
- The user explicitly requests Chinese

The workflow instructions themselves (this document) stay in English. Commands and code examples are language-neutral.

---

## The 4 required blocks: A / B / C / E

Run in order. **Every block must output concrete commands and their results as evidence.** Run parallelizable bash in a single invocation.

| Block | What to verify |
|---|---|
| **A — Change sites** | New strings are present; old strings are fully removed; no missed variants |
| **B — Boundary conditions** | Threshold neighbors, null, empty, and extreme values don't blow up |
| **C — File integrity** | Syntax is legal; tags / brackets are balanced |
| **E — Call-chain context** | Upstream/downstream of the modified code is still consistent |

---

### Block A — Change-site verification ("did the edit actually land?")

1. First, have the user (or yourself) recall **which files and which positions were changed this round.**
2. For each change:
   - `grep -n '<new key string>' <file>` — confirm new content is present
   - `grep -n '<old key string>' <file>` — confirm old content is gone (should be 0 matches)
3. If this was a "global replacement / N similar edits":
   - **Re-grep the entire file for every variant** of the original string to catch misses.
   - Typical miss patterns: `404 / 5xx` → forgot `404+5xx` / `404 + 5xx` / `404/5xx`.
   - Command: `grep -nE 'pattern1|pattern2|pattern3' <file>`
4. Show the user a diff overview (if backups or git exist):

```bash
diff -u <file>.bak.YYYYMMDD <file> | grep -E '^[+-]' | grep -v '^[+-]{3}'
# or
git diff <file>
```

### Block B — Boundary condition simulation ("will extreme values blow up?")

For numeric changes, write a quick Python snippet:

```python
for v in [0, threshold-1, threshold, threshold+1, typical, upper_bound]:
    # simulate the code logic, print behavior for each
```

Example: if you changed `if (s5xx > 10)`, enumerate `s5xx ∈ {0, 5, 10, 11, 50, 101, 1001}` — show trigger / no-trigger + severity for each.

For string changes, ask yourself:

- Does an empty value render oddly?
- Will null / undefined cause NaN downstream?
- Will an overflow-length string break the layout?
- Special characters in username / URL (quotes, angle brackets) — XSS risk?

### Block C — File integrity ("did the syntax / structure break?")

By file type:

**Python:**

```bash
python3 -c "import ast; ast.parse(open('xxx.py').read()); print('✅ valid')"
```

**HTML:**

```python
h = open('xxx.html').read()
import re
divs_o = len(re.findall(r'<div\b', h))
divs_c = len(re.findall(r'</div>', h))
# script, style, a, span tags — same pattern
print(f'div: {divs_o}/{divs_c}')
# template literal backticks must be even
print(f'backticks: {h.count(chr(96))}')
```

**JSON:**

```bash
python3 -c "import json; json.load(open('xxx.json')); print('✅ valid JSON')"
```

**YAML:**

```bash
python3 -c "import yaml; yaml.safe_load(open('xxx.yml')); print('✅ valid YAML')"
```

**Shell:**

```bash
bash -n xxx.sh && echo '✅ syntax OK'
```

**Systemd unit:**

```bash
systemd-analyze verify ~/.config/systemd/user/xxx.service 2>&1
```

### Block E — Call-chain context ("is the modified code still consistent with its surroundings?")

**This block is what the user cares most about** — the "call chain" (流程链条). When modifying code, don't just look at the changed line; walk the **full upstream + downstream path** through that line.

#### 4 required directions

**1. Data flow upstream (where does this variable / field come from?)**

- For each variable used in the change, how is it initialized in the same scope?
- If it comes from data.json / API / env var, does the source format match?
- Example: changed the KPI showing `s5xx` → trace `const s5xx = ...` back to the source (`nginx.s5xx`); ask "what if the nginx object is missing? what if s5xx is a string rather than a number?"

**2. Data flow downstream (who uses this thing?)**

- `grep -n <modified variable/function>` across the project — who else references it?
- Changed a KPI formula → check derived vars like `nginxErrCount`.
- Changed a function signature → every call site must be updated (or runtime TypeError).
- Changed a data.json field → every front-end reader of that field must be checked.

**3. Horizontal chain (is the full flow still working?)**

- **Scheduled-task changes:**
  - systemd timer still active? `systemctl --user list-timers <name>`
  - Wrapper script can still locate the binary?
  - Rules in the prompt file still in place?
  - Memory file still present? Will the next run read it?
  - Linger file `/var/lib/systemd/linger/<user>` still there?
- **Web publish changes:**
  - Did the publish script run?
  - Local md5 == remote md5? (`curl` and compare)
  - Cacheless DOM fetch + visual verify (Playwright)
- **Database / schema changes:**
  - Can old data be consumed by the new code?
  - Can new data written by new code be read by old clients?

**4. Time / state consistency**

- Date / timezone changes: list current JST / UTC / CST times and confirm which one the user means.
- Deduplication / lock / dedup changes: enumerate all dedup conditions; simulate concurrency / retry.
- Caching changes: list cache layers (local / Cloudflare / browser); when does each invalidate?

#### Typical "chain not fully checked" pitfalls

| What changed | Easy-to-miss chain |
|---|---|
| One line inside a function | Callers pass wrong argument types |
| A JSON field name | Front-end readers, SQL queries, export scripts |
| A scheduled-task prompt | Claude's auto-memory retains old rules (conflict) |
| A color / threshold | Matching legend / tooltip / chart caption not updated |
| A file's charset | Downstream reader's encoding assumption |
| Publish logic | CDN cache TTL keeps the old version alive |

---

## Output format (shown to the user)

### Recommended structure

**English:**

```markdown
# Full verification passed ✅  (or ⚠️ N issues found)

| Check | Result | Evidence |
|---|---|---|
| Change-site A1 | ✅ | grep found 3 occurrences of the new string |
| Change-site A2 | ✅ | 0 residual old-string matches |
| Boundary s5xx=0 | ✅ | returns green |
| Boundary s5xx=11 | ✅ | yellow + alert triggered |
| HTML structure | ✅ | div 807/807 balanced |
| Chain: callers | ✅ | 3 call sites, signatures consistent |
| Chain: data source | ✅ | nginx.s5xx field exists and is numeric |
| Chain: timer | ✅ | active, next 2h26m |

## What happens on the next execution (full walkthrough)
1. ...
2. ...
```

**中文:**

```markdown
# 全量核验通过 ✅（或者 ⚠️ 发现 N 个问题）

| 检查项 | 结果 | 证据 |
|---|---|---|
| 改动点 A1 | ✅ | grep 出 3 处新字符串 |
| 改动点 A2 | ✅ | 0 处残留旧字符串 |
| 边界 s5xx=0 | ✅ | 返回绿色 |
| 边界 s5xx=11 | ✅ | 黄色 + alert 触发 |
| HTML 结构 | ✅ | div 807/807 平衡 |
| 链条：调用方 | ✅ | 3 处调用签名一致 |
| 链条：数据源 | ✅ | nginx.s5xx 字段存在且为数字 |
| 链条：定时器 | ✅ | active, next 2h26m |

## 下次执行时会发生的事（完整推演）
1. ...
2. ...
```

### Absolutely DO NOT

- ❌ Claim "all checked" without showing the command output
- ❌ Skip boundary conditions — the user invoked the skill *because* they worry about those
- ❌ Equate "no issues found" with "no issues" — admit the scope limit
- ❌ Modify code directly — this skill is **read-only verification**. Report findings and let the user decide
- ❌ Compress 10 checks into one "all OK" — the user will re-ask when they can't see evidence
- ❌ Look only at the changed line — **walk the upstream and downstream chain**

### If issues are found

- Highlight clearly (red / ⚠️)
- State in what scenario the issue triggers
- State the impact scope (whole-page crash / single field blank / alert only)
- Suggest a fix but **wait for user confirmation** before changing code

---

## Extra checks for specific scenarios

| User just did | Pay special attention to |
|---|---|
| Bulk string replacement | Grep every variant of the old string (whitespace, half/full-width, different separators) |
| Scheduled-task config change | `systemctl list-timers` + linger + prompt/memory consistency + full dry-run of the next trigger |
| prompt / memory edit | What Claude will read next time, and whether it conflicts with old memory |
| Frontend + publish | Full chain: source file → publish → remote → CDN cache → browser cache |
| Keys / permissions | `chmod` bits, owner, group, sudo capability, readability by other users |
| Schema change | Old and new field data both consumable; front-end / export / SQL all compatible |
| Timezone / date | List Beijing / JST / UTC current times, identify which the user sees |
| Function-signature change | Grep every call site project-wide, confirm all are updated |

---

## Self-check: was this thorough-check actually thorough?

After running, ask yourself:

1. If the user asks "are you sure?" again, do I have any new info to offer? → If no, not thorough enough.
2. Did I write any "speculation" as "confirmation"? → Must be measured, not inferred.
3. Did I actually walk the upstream and downstream chain of the change? → Not just the line itself.
4. If this blows up tomorrow, can I trace back from my output to the blind spot? → If yes, passing grade.
