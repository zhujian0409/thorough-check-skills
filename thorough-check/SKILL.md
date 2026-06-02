---
name: thorough-check
description: 用于彻底复核刚修改过的代码或配置，覆盖改动点、边界条件、文件完整性、上下游调用链，以及任何参数或代码改动引发的业务语义变化、历史数据兼容、清理任务、回调/事件、幂等和状态迁移风险。用于实际复核时，默认启动独立 agent 做并行影响面扫描和最终反证复核；token 成本不是省略 agent 的理由。没有真实命令输出证据时不宣称“OK”。触发语包括“改完了帮我再仔细检查一遍”“各种边界都 ok 吗”“流程链条都对吗”“能确定不会出问题吗”“这个参数改了会不会影响上下游”等。
allowed-tools: Read Grep Bash Task
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

## Multi-agent cross-check

When this skill is invoked for an actual code/config verification task, treat that invocation as standing user authorization to use reviewer agents. Start reviewer agents by default whenever the current runtime and tool policy allow subagents. Token cost is not a reason to skip them. The parent agent still owns the final judgment; subagents are used to find missed impact, challenge assumptions, and verify the draft result.

Only skip subagents when:

- The current environment has no subagent capability, or tool policy blocks it.
- The user explicitly says not to use agents/subagents for this run.
- The task is not a verification run at all, such as the user only asking what this skill means.

Default pattern:

1. Start one read-only **impact reviewer agent** near Block 0. Ask it to independently search for affected files, callers, old/new states, jobs, cache, permissions, data contracts, and rollback risks.
2. Start one read-only **final reviewer agent** before the final answer. Give it the parent draft findings and ask it to disprove "passed" claims, find missing evidence, and challenge skipped blast-radius areas.
3. While agents work, the parent agent continues Blocks 0/A/B/C/E locally. Do not wait idly unless the next step is blocked by an agent result.
4. For broad or high-risk changes, start additional focused read-only agents with disjoint scopes, such as data/state compatibility, async jobs/cache, auth/API, frontend/publish, or rollback/deploy.
5. Agent handoff must be narrow and evidence-based: tell the subagent which repo/files changed, require read-only commands, require file/line references or command output, and ask for a short findings table.
6. All agents must stay read-only: no code edits, no data mutation, no deploy, no cleanup scripts unless the user explicitly authorizes a separate fixing task.
7. Agent conclusions are not proof unless backed by command output, file references, query results, screenshots, or reproducible test output.
8. Merge agent findings into the final report. Include which agent checks were run, what they found, what the parent accepted or rejected, and any unresolved disagreement.
9. If agents disagree, preserve the disagreement in the final report and mark that area unresolved until evidence decides it.

If subagents cannot be spawned because the runtime or tool policy blocks them, or because the user explicitly opted out, do a sequential fallback and explicitly say: "No independent agent cross-check was run; reason: ...". Never imply agent review happened when it did not.

## The required blocks: 0 / A / B / C / E

Run in order. **Every block must output concrete commands and their results as evidence.** In Codex, parallelize independent reads/searches with the parallel tool when available; otherwise run the smallest clear commands needed.

| Block | What to verify |
|---|---|
| **0 — Impact map** | What changed, what it touches, and what files/contracts could be affected |
| **A — Change sites** | New strings are present; old strings are fully removed; no missed variants |
| **B — Boundary conditions** | Threshold neighbors, null, empty, and extreme values don't blow up |
| **C — File integrity** | Syntax is legal; tags / brackets are balanced |
| **E — Call-chain context** | Upstream/downstream of the modified code is still consistent |

Use stable labels in every report so the chain is easy to audit later:

- **Old value / New value**: for parameter, text, enum, status, permission, feature-flag, route, or schema changes, explicitly name the previous value and the new value before judging behavior.
- **Upstream / Downstream**: in Block E, explicitly label where changed data comes from and who consumes it. Do not rely on vague "callers checked" wording when a reader needs to trace the chain.
- **Excluded files**: in dirty Git/SVN working copies, explicitly list generated, temporary, QA, report, cache, build, or local-only files to exclude from commit/deploy scope.
- **Next run**: for scheduled jobs, automations, prompt/memory edits, and publish flows, explicitly describe what the next execution will read and do.

---

### Block 0 — Impact-map first ("how big is the blast radius?")

Before checking details, build an explicit impact map. A tiny edit can be high risk if it changes a shared field, enum, status, endpoint, permission rule, scheduled task, queue message, cache key, config value, or external integration contract.

Run the smallest available commands that reveal scope:

```bash
git diff --name-only
git diff --stat
git diff -- <changed-file>
rg -n "<changed function|field|status|route|config key>" .
rg -n "<old value>|<new value>|<enum/status variants>" .
```

If the repo is SVN or has no git metadata, use its equivalent (`svn diff`, `zsvn st`) plus `rg`.

Classify the change before continuing:

| Change type | Required extra attention |
|---|---|
| Function signature / return shape | Every caller, tests, mocks, API wrappers, background jobs |
| Field name / schema / enum / status | Old data, new data, exports, imports, SQL/Mongo queries, admin pages |
| Config / feature flag / parameter | Defaults, env overrides, deploy config, rollback, mixed old/new processes |
| Endpoint / route / payload | Frontend callers, mobile/old clients, auth, validation, logs, webhooks/events |
| Permission / scope / role | Positive cases, negative cases, cross-role leakage, public/anonymous behavior |
| Async job / queue / cron | Retry, dedup, lock, partial failure, next scheduled run, old queued messages |
| Cache / CDN / local storage | Cache keys, TTL, invalidation, stale readers, browser/server mismatch |
| External provider / integration | Sandbox vs prod, callback mapping, rate limits, timeout/retry, idempotency |
| Deletion / cleanup | Who still imports/reads it, packaging, deploy scripts, docs, generated files |
| UI/display text | Matching legend, tooltip, export label, screenshot/mobile layout, i18n copy |

Output a short blast-radius table before saying whether the change is safe:

```markdown
| Area | Files / commands checked | Risk | Evidence |
|---|---|---|---|
| Direct diff | ... | Low/Medium/High | ... |
| Callers/readers | ... | ... | ... |
| Data compatibility | ... | ... | ... |
| Jobs/events/cache | ... | ... | ... |
| Rollback/mixed deploy | ... | ... | ... |
```

If this map is incomplete because the repo, database, server, or dependency is unavailable, say exactly which part is unverified and do not call the whole check "passed".

### Block A — Change-site verification ("did the edit actually land?")

1. First, have the user (or yourself) recall **which files and which positions were changed this round.**
2. For each change:
   - `grep -n '<new key string>' <file>` — confirm new content is present
   - `grep -n '<old key string>' <file>` — confirm old content is gone (should be 0 matches)
   - Write a short **Old value / New value** row for every changed threshold, config, label, enum/status, route, or schema field, even when the values are obvious from the diff.
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

#### Required directions

**1. Data flow upstream (where does this variable / field come from?)**

- For each variable used in the change, how is it initialized in the same scope?
- If it comes from data.json / API / env var, does the source format match?
- Example: changed the KPI showing `s5xx` → trace `const s5xx = ...` back to the source (`nginx.s5xx`); ask "what if the nginx object is missing? what if s5xx is a string rather than a number?"
- Label this section **Upstream** in the final report and include file/function/source names, not just prose.

**2. Data flow downstream (who uses this thing?)**

- `grep -n <modified variable/function>` across the project — who else references it?
- Changed a KPI formula → check derived vars like `nginxErrCount`.
- Changed a function signature → every call site must be updated (or runtime TypeError).
- Changed a data.json field → every front-end reader of that field must be checked.
- Label this section **Downstream** in the final report and include each consumer/caller/read path checked.

Also search by concept, not just exact symbol. Many breakages hide behind aliases:

- Route aliases, controller/action names, template variable names, translated labels.
- DB column names, JSON keys, Mongo field names, SQL aliases, export headers.
- Enum/status values in different spelling/case (`active`, `ACTIVE`, `1`, `enabled`).
- Frontend/backend duplicate constants, generated SDKs, mocks, fixtures, docs, tests.
- Shell scripts, cron wrappers, deploy scripts, import/export scripts, one-off repair scripts.

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

For multi-file flows, explicitly walk the main path end to end:

```text
entrypoint -> validation/auth -> business logic -> storage/write -> async side effects -> read/display/export -> monitoring/logs
```

At each step, name the exact file/function checked. If a step is outside the current repo, mark it as external/unverified instead of silently skipping it.

**4. Time / state consistency**

- Date / timezone changes: list current JST / UTC / CST times and confirm which one the user means.
- Deduplication / lock / dedup changes: enumerate all dedup conditions; simulate concurrency / retry.
- Caching changes: list cache layers (local / Cloudflare / browser); when does each invalidate?

**5. Existing-state / migration consistency for behavior changes**

When a change alters business semantics, not just display text or formatting, treat it as a state migration. Examples: old mode -> new mode, manual step -> automatic step, pending -> completed, soft delete -> hard delete, async processing -> sync processing, optional field -> required field, one provider/channel/rule -> another.

Verify with commands and concrete evidence:

- State inventory: list all existing statuses / enum values / DB records affected by the old and new behavior.
- In-flight records: what happens to records created before deploy but processed after deploy?
- Old clients / old jobs: can old app versions, cron jobs, callbacks, retries, exports, admin tools, and support scripts still consume the new state?
- Cleanup / reconciliation: identify scheduled jobs, queues, timeout handlers, cancel/expire tasks, reconciliation jobs, and manual repair scripts that used to assume the old behavior.
- Idempotency and retry: simulate duplicate webhook, duplicate submit, network timeout after success, retry after failure, and partial write scenarios.
- Rollback path: if code rolls back after some new-state records were created, can old code read or repair them?
- Reporting / audit / user-visible behavior: confirm dashboards, exports, reconciliation, audit logs, notifications, and customer/admin/support messages match the new behavior.
- Mixed deployment: if multiple services deploy at different times, list which combinations are safe and which are not.
- Backfill / migration decision: explicitly state whether old records need migration, cleanup, one-time scripts, or no action, and show the query / grep evidence.

For any state-machine or lifecycle change, also verify:

- Every old state, new state, intermediate state, terminal state, timeout state, and failure state has an explicit next action.
- Historical records created under the old behavior still have a valid path: complete, cancel, expire, retry, migrate, ignore, or manual handling.
- New records created under the new behavior cannot be reprocessed incorrectly by old jobs, old callbacks/events, or retries.
- External callbacks, async events, scheduled jobs, queues, admin actions, and manual repair scripts are still mapped to internal states correctly.
- Empty, duplicate, failed, partial, delayed, out-of-order, and already-processed cases are covered.
- Customer/admin/support wording no longer describes the old behavior if the system now behaves differently.
- Monitoring and alerts distinguish old-state and new-state meanings if those meanings differ.

**6. Regression surface and test selection**

Do not only run the easiest syntax check. Pick verification proportional to risk:

- Low-risk text/style change: changed file syntax + affected render/output check.
- Shared helper or function change: direct unit tests + every caller signature/search check.
- API/schema/status change: request/response sample, old-data sample, new-data sample, negative auth/validation sample.
- Cron/queue/job change: dry-run or next-run simulation, duplicate/retry scenario, log path, lock/dedup check.
- Frontend workflow change: browser smoke test for desktop/mobile or at least DOM/render check if browser is unavailable.
- Production/config/deploy change: local config, deployed config, secrets/env names, restart/reload path, rollback path.

If tests cannot run, show the command attempted and the blocker. Then list the highest-value remaining tests instead of declaring success.

#### Typical "chain not fully checked" pitfalls

| What changed | Easy-to-miss chain |
|---|---|
| One line inside a function | Callers pass wrong argument types |
| A JSON field name | Front-end readers, SQL queries, export scripts |
| A scheduled-task prompt | Prior assistant memory / persisted instructions retain old rules (conflict) |
| A color / threshold | Matching legend / tooltip / chart caption not updated |
| A file's charset | Downstream reader's encoding assumption |
| Publish logic | CDN cache TTL keeps the old version alive |
| A feature flag / parameter | Default values, env overrides, old workers, rollback, mixed deployment |
| A status / enum | Historical records, pending records, terminal records, retry/cleanup jobs |
| A permission check | Admin/user/public roles, stale sessions, menu entrypoints, API direct access |
| A helper used "everywhere" | Generated files, tests/mocks, CLI scripts, imports in sibling modules |
| A delete/rename | Packaging, deploy scripts, docs, references, dynamic imports, old cron commands |

---

## Output format (shown to the user)

### Recommended structure

**English:**

```markdown
# Full verification passed ✅  (or ⚠️ N issues found)

| Check | Result | Evidence |
|---|---|---|
| Agent cross-check | ✅ | impact reviewer checked callers/jobs/cache; final reviewer found no unsupported pass claims |
| Agent fallback | ⚠️ | No independent agent cross-check was run; reason: current runtime has no subagent tool |
| Impact map | ✅ | 5 changed files; 11 callers checked; no external readers found |
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
| Agent 交叉复核 | ✅ | impact reviewer 已查调用方/任务/缓存；final reviewer 未发现无证据通过项 |
| Agent fallback | ⚠️ | No independent agent cross-check was run; reason: 当前环境没有 subagent 工具 |
| 影响面地图 | ✅ | 5 个改动文件；已查 11 个调用方；未发现外部读取方 |
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
- ❌ Treat a small diff as a small risk before building an impact map
- ❌ Search only exact variable names when the same concept may appear as route names, DB fields, labels, enum values, or scripts
- ❌ Ignore old data, old queued jobs, old clients, stale cache, or rollback just because the new code path works once
- ❌ Claim independent agent review happened when no subagent was actually started or when its result was not incorporated

### If issues are found

- Highlight clearly (red / ⚠️)
- State in what scenario the issue triggers
- State the impact scope (whole-page crash / single field blank / alert only)
- Suggest a fix but **wait for user confirmation** before changing code

---

## Extra checks for specific scenarios

| User just did | Pay special attention to |
|---|---|
| Small code change with unclear blast radius | Start with Block 0; list changed files, symbols, callers, readers, jobs, configs, and tests before judging risk |
| Bulk string replacement | Grep every variant of the old string (whitespace, half/full-width, different separators) |
| Scheduled-task config change | `systemctl list-timers` + linger + prompt/memory consistency + full dry-run of the next trigger |
| prompt / memory edit | What Codex will read next time, and whether it conflicts with old memory |
| Frontend + publish | Full chain: source file → publish → remote → CDN cache → browser cache |
| Dirty Git/SVN working copy | Separate commit/deploy scope from local noise; explicitly list included files and **exclude temp/generated/QA/report/cache/build files** |
| Keys / permissions | `chmod` bits, owner, group, sudo capability, readability by other users |
| Schema change | Old and new field data both consumable; front-end / export / SQL all compatible |
| State-machine / lifecycle behavior change | Treat as state migration: historical records, in-flight records, callbacks/events, retries, complete/cancel/expire/retry/manual paths, cleanup jobs, reconciliation, rollback, and mixed deployments |
| One parameter changes business behavior | Do not stop at the parameter. Trace the full business semantics before and after the change, then enumerate every old/new state combination and the next action for each |
| Auth / role / scope change | Positive and negative role cases, stale session/cookie, menu entry, direct API access, public/anonymous behavior |
| API payload / route change | Frontend callers, old clients, validation, auth, sample request/response, logs, callbacks/events |
| Queue / retry / async event change | Existing queued messages, duplicate delivery, out-of-order events, lock/dedup, partial writes, failure recovery |
| Cache behavior change | Key composition, TTL, invalidation path, stale readers, browser/server/CDN layers |
| Delete / rename / move files | Imports, dynamic references, packaging/deploy scripts, docs, generated files, cron commands |
| Timezone / date | List Beijing / JST / UTC current times, identify which the user sees |
| Function-signature change | Grep every call site project-wide, confirm all are updated |

---

## Self-check: was this thorough-check actually thorough?

After running, ask yourself:

1. If the user asks "are you sure?" again, do I have any new info to offer? → If no, not thorough enough.
2. Did I write any "speculation" as "confirmation"? → Must be measured, not inferred.
3. Did I actually walk the upstream and downstream chain of the change? → Not just the line itself.
4. Did I search for concept aliases, not only the exact string I edited?
5. Did I explicitly state the blast radius: what is affected, what is checked and unaffected, and what remains unverified?
6. If subagents were available and the user did not opt out, did I start independent reviewers and merge their findings? If not, did I state the runtime/tool-policy blocker?
7. If this is a business behavior change, did I explicitly account for old records, in-flight records, new records, retries, callbacks/events, cleanup, rollback, and mixed deployments?
8. If this blows up tomorrow, can I trace back from my output to the blind spot? → If yes, passing grade.
