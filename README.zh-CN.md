# thorough-check

**语言：** [English](README.md) | 中文

[@zhujian0409](https://github.com/zhujian0409) 写的一个 [Claude Code](https://docs.claude.com/en/docs/agents-and-tools/claude-code/overview) skill：对刚做完的代码/配置改动做彻底核验，默认启动只读 reviewer agent 做交叉复核，拿不出命令输出当证据就拒绝说"OK"。

## 干什么用

改完之后问一句"仔细检查一下"，平时能拿到的答复通常是客气的"看起来没问题"——然后一小时后线上就炸了。

`thorough-check` 强制跑证据优先的核验，每一块**没有具体命令输出**就不许过关：

| 块 | 核验内容 |
|---|---|
| **0 — 影响面地图** | 改了什么、牵扯哪些文件/契约/任务/缓存/数据状态，先画 blast radius |
| **A — 改动点** | 新字符串存在；旧字符串彻底清干净；变体没漏（比如 `404/5xx` vs `404+5xx` vs `404 + 5xx`） |
| **B — 边界条件** | 阈值相邻、空值、null、超长字符串——极端输入是否还能正常渲染 |
| **C — 文件完整性** | 语法合法；标签/括号平衡；JSON/YAML/Python/HTML 能干净 parse |
| **E — 调用链上下文** | 上游数据源格式还对得上；下游每个调用方都兼容；状态迁移、历史数据、回滚、重试、队列、发布和缓存风险都要交代清楚 |

真实复核任务里，只要运行时支持，skill 默认启动只读 reviewer agent：

- **impact reviewer**：并行寻找漏掉的影响面；
- **final reviewer**：最终反证主报告里有没有无证据通过、漏边界、漏链路。

token 成本不是跳过 reviewer 的理由。如果运行时不能启动 subagent，最终报告必须明确说明原因。

每一条都会在最终的表格里给出**检查项 / 结果 / 证据**三列。没有具体 grep/diff/parse 输出的那一条会被打 ⚠️ 并要求你给出真实命令，不允许用"看起来还行"糊过去。

输出语言自动识别：默认英文，当对话主要是中文时切中文。

## 什么时候调用

这个 skill **会自动触发**——当你用下面这类"我对刚才的改动不放心"式表达时：

**中文:**

- "改完了帮我再仔细检查一遍"
- "各种边界都 ok 吗"
- "流程链条都对吗"
- "能确定不会出问题吗"
- "仔细反复的检查 确定都没问题是吧"

**English:**

- "Double-check everything carefully"
- "Are all the edge cases OK?"
- "Is the entire call chain consistent?"
- "Verify this change won't break anything"

和 `stakeholder-writeup` 不一样，这个是**故意让它能自动被触发**——skill frontmatter 里**不设** `disable-model-invocation`，Claude 能够自己识别你的担忧表达。

## 安装

clone 本仓库，把 skill 目录拷到你的 Claude Code 用户级 skill 目录：

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/zhujian0409/thorough-check-skills.git
cp -r thorough-check-skills/thorough-check ~/.claude/skills/
```

或者保留本仓库，用软链接（这样 `git pull` 就能直接更新 skill）：

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/zhujian0409/thorough-check-skills.git
ln -s "$(pwd)/thorough-check-skills/thorough-check" ~/.claude/skills/thorough-check
```

放到 `~/.claude/skills/` 下的 skill 会被 Claude Code 自动注册。下一次会话里，你说出触发短语就会直接被接住。

## 设计理念

1. **要证据，不要感觉。** 任何结论都必须有具体命令输出支撑，宁愿给你列 20 条啰嗦的 ✅/❌，也不给一句不带证据的"全都 OK"。
2. **走完整条链路，不只看被改的那一行。** 大多数回归不是来自你改动的那一行本身，而是来自上游的数据源或下游被遗忘的调用方。skill 强制你把两头都走一遍。

完整清单、默认多 agent 交叉复核规则、常见"链条没查全"的坑点、以及 skill 每次核验完对自己的自检，见 [SKILL.md](thorough-check/SKILL.md)。

## License

MIT——见 [LICENSE](LICENSE)。
