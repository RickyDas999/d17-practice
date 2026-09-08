# Day 17 CORE — Fix the Agent That Guesses

A single-tool order-support agent built on the Claude Agent SDK. The SDK runs the
agentic loop; the rules live in a Skill, not in the system prompt.

## Files

| Path | Role |
|---|---|
| `agent.py` | The tool (`get_order_status`), fake order data, the `ClaudeAgentOptions` block, and the message-stream reader |
| `.claude/skills/order-support/SKILL.md` | The behaviour rules: always look up, never guess, admit not-found, keep it short |

## Run

```bash
uv run python agent.py "What's the status of order A1002?"
uv run python agent.py "Where is order Z9999?"      # not in the data — must say so, not invent
```

Auth comes from the logged-in `claude` CLI — no `ANTHROPIC_API_KEY` / `.env` needed.

Real order IDs in the fake data: `A1001` (delivered), `A1002` (in transit), `A1003` (processing).

## The bug this exercise is about

`skills=["order-support"]` only *names* the Skill. The SDK spawns the `claude`
CLI as a subprocess, and that subprocess has to **find `SKILL.md` on disk** and
inject it into the system prompt before the first model call. Two options make
that possible, and the assignment's starter block had neither:

```python
cwd=os.path.dirname(os.path.abspath(__file__)),  # where the CLI looks for .claude/skills/
setting_sources=["project"],                     # permission to read project config
```

Without `cwd`, the CLI looks relative to wherever you launched the process, not
where the script lives. A filter (`skills=[...]`) that matches zero discovered
skills is **not an error** — so the agent starts fine and just answers without
its rules. The only signal is line 1 of output: `Skill 'order-support' loaded: False`.

> Version note: in `claude-agent-sdk` 0.2.152, a non-empty `skills=[...]` list
> auto-defaults `setting_sources=["user","project"]`, so the still-live half of
> the bug is `cwd`. Both lines are kept — correct and version-proof.

## Success criteria

- Startup prints `Skill 'order-support' loaded: True`
- A real order returns its real status/carrier/ETA, with a `[tool]` line proving the tool was called
- A fake order ID produces an honest "not found", not an invented order

---

## Part B — Self-Test

**1. An agent runs with no errors but keeps inventing data its Skill forbids.
What single line of output tells you the Skill didn't load, and what two options
fix it?**

The startup line `Skill '<name>' loaded: False`. It means the Skill was named but
never discovered so a current working directory needs to be set (cwd) with the skill location and setting_sources=['project] to give Claude the permission to read the skill within the project directory. 

**2. Why is a silent failure (wrong answer, no error) more dangerous in
production than a loud one (a crash)?**

A silent failure in production does not tell anyone that it is ocurring so it may be duplicated for very long until somebody directly looks for it. A loud crash will immediately page an engineer and disallow further issues that may occur in a production setting. 