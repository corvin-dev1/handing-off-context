# handing-off-context

A Claude Code skill for ending a long, token-heavy conversation cleanly and
resuming it in a fresh chat — without re-reading everything from scratch.

## The problem

Long sessions accumulate full-file reads, repeated tool output, and stale
context. Past a point, every turn in that chat costs more tokens than it
should — but just starting a new chat and saying "continue where we left
off" forces the new chat to re-derive everything the hard way (reading
files end to end again, asking you what happened).

## What this skill does

When invoked, it produces a **compact handoff prompt** (state, next step,
key files/symbols, active constraints — never raw transcript or full file
dumps) that you paste into a new chat. The new chat resumes instantly with
the right context, and is instructed to prefer targeted lookups (a code
graph, `grep`) over reading whole files again.

## Install

Copy the skill into your Claude Code skills directory:

```bash
git clone https://github.com/corvin-dev1/handing-off-context.git ~/.claude/skills/handing-off-context
```

Or as a plugin:

```bash
claude plugin install corvin-dev1/handing-off-context
```

## Usage

In a chat that's gotten long or expensive:

```
/handing-off-context
```

or just ask Claude to "hand this off to a new chat" — the skill triggers
on that intent. It produces a ready-to-paste prompt for the new chat.

## Why this exists

Built after noticing a session burn far more tokens than usual because it
read large source files in full instead of using an available code-graph
lookup. The fix wasn't just "read less" for that one session — it's a
reusable handoff discipline: summarize state, not history, and hand the
new chat the rule to look before it reads.

## License

MIT
