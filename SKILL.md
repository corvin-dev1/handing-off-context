---
name: handing-off-context
description: Use when a conversation has grown long (heavy token usage, many full-file reads, repeated large context) and work needs to continue in a fresh chat without re-reading everything from scratch — produces a persistent on-disk handoff record plus a compact ephemeral prompt, and enforces graph/grep-first lookups over full-file reads in the new chat.
---

# Handing Off Context

## Overview

A long-running session accumulates full-file reads, repeated tool output, and stale context. Past a point, continuing in the same chat costs more tokens per turn than starting fresh — but starting fresh naively means re-deriving everything the hard way (reading files end to end again). This skill produces **two artifacts, not one**:

1. A **persistent handoff file**, written to disk in the project (`docs/handoffs/` or similar) — the durable narrative of what was done, what was decided, and why. It accumulates across handoffs and survives after the chat that wrote it is gone.
2. An **ephemeral handoff prompt**, pasted into the new chat — points at the persistent file and the codebase instead of restating them, carries the one concrete next step, and stays disposable.

Splitting the two matters: the ephemeral prompt alone (just a compressed summary of the chat) is what most naive handoffs do, and it loses the *why* behind past decisions once that chat is gone. The persistent file keeps that trail — so if a decision gets questioned three sessions later, the reasoning is still on disk, not lost in a chat transcript nobody can search. It also sets the rule that the new chat must prefer indexed/targeted lookups (a code graph, `grep`, `graphify query`) over reading whole files.

## When to Use

Symptoms in the CURRENT (old) chat:
- Several full-file `Read` calls on large files (1000+ lines) already happened
- The conversation has been summarized/compacted more than once
- You notice yourself about to re-read a file you already read earlier just to "remind yourself"
- The user is running close to a usage/token limit and work isn't finished

Do NOT use for short conversations, or when the task is nearly done — finish first.

## What to Do in the OLD Chat

1. Tell the user plainly that context is getting heavy and a fresh chat will be more efficient.
2. Write or append to the **persistent handoff file** on disk (see template below) — this is the durable record, not a scratch note.
3. Produce the **ephemeral handoff prompt** (see template below) for the user to paste into the new chat. It references the persistent file by path; it does not repeat its content.
4. Do not dump raw transcript, full file contents, or tool output into either artifact — summarize state and decisions, not history.

## Persistent Handoff File

Location: `docs/handoffs/<project-or-task-name>.md` (or the project's existing convention for durable notes, if one exists — don't invent a second location).

Append a new dated section each time a handoff happens; never overwrite prior sections. Structure per entry:

```
## <date> — <short task name>

**Qué se hizo:** <what was actually built/fixed/decided this segment — branch/diff state, commands/tests already run>

**Por qué:** <the reasoning — especially rejected paths and non-obvious decisions, so they don't get relitigated>

**Pendiente:** <what's still open, the exact next acceptance check, and what's blocking it>

**Artefactos:** <references/IDs to logs, diffs, screenshots — link out, never paste raw output in here>
```

This file is the thing that survives after the chat is gone — write it as if a future session (or a different person) has to understand a decision with zero other context. **Write to it at the end of every session that made real progress, not only when a handoff is explicitly requested** — an unwritten segment is a segment that's lost once the chat closes.

Keep it from rotting into a junk drawer: once it holds more than ~5 dated entries, fold the older ones into a one-paragraph summary at the top (or move them to an `archive.md` next to it) — a growing pile of stale entries is exactly what the next chat has to skim past to find what's current.

## Ephemeral Handoff Prompt Template

Keep it under ~300 words. Structure:

```
## Handoff: <short task name>

**Registro persistente:** <path to the persistent handoff file — read it first, don't ask to be told what it says>

**Próximo paso concreto:** <the single next action, not a list of options>

**Archivos/símbolos clave (NO leer completos — usar grep/graphify primero):**
- <file:line or symbol> — <why it matters>

**Herramientas a usar antes de leer archivos:**
- graphify: `graphify query "<question>" --graph <path-to-graph.json>`
- grep/ripgrep for exact symbols before opening a file

**Skills relevantes a activar:** <skill names, only if truly applicable — don't list generically>

**Restricciones activas:** <any hard rules still in force, e.g. "no marcar X completo sin confirmación en vivo del usuario">

**No ejecutar ni modificar nada todavía:** solo confirma que entendiste el estado (del registro persistente) y pregunta qué se quiere hacer antes de tocar código o correr comandos.
```

The last line is not optional boilerplate — always include it verbatim. Without it, the new chat's own "Auto Mode" bias (make the reasonable call and keep going instead of stopping to check) can read the handoff's "próximo paso concreto" as authorization to act immediately, before the user has actually said go.

**No chat tool (Claude web, no Code/hooks):** collapse the ephemeral prompt into the Project's custom instructions instead of pasting it each time — it becomes always-on, and the only artifact you still move manually is the persistent file (paste it at the start of each new chat). Same split, one less step.

## The Rule for the NEW Chat

State this explicitly in the handoff prompt, not just as a note to self:

**Before opening any source file, try a targeted lookup first** (graph query for the exact symbol, or `grep`/`rg` for the exact string). Read the whole file only when the targeted lookup doesn't resolve it, and even then prefer `Read` with `offset`/`limit` over reading files end to end when the file is large and the relevant section is known.

## Quick Reference

| Situation | Do |
|---|---|
| About to re-read a file already read this session | Don't — recall or grep the specific part instead |
| Need to find where a symbol lives | Graph query / grep first, `Read` second |
| File is large (1000+ lines) and you need one function | `Read` with `offset`/`limit`, not the whole file |
| Conversation flagged as heavy by user or by repeated compaction | Offer a handoff prompt now, don't wait |

## Common Mistakes

- Only writing the ephemeral prompt and skipping the persistent file — the *why* behind decisions dies with the chat, and the next-next handoff has nothing durable to point to.
- Handoff prompt that pastes large chunks of prior transcript — defeats the purpose, costs tokens in the new chat too.
- Vague "continue where we left off" with no concrete next step — forces the new chat to re-derive state by asking questions.
- Listing every skill available instead of the 1-2 actually relevant ones.
- Forgetting to name the exact graph path — the new chat then has to search for it.
- Overwriting the persistent file instead of appending — destroys the trail of past decisions the file exists to preserve.
- Pasting raw logs/diffs/screenshots into the persistent file instead of a reference — turns the map into the warehouse, and bloats every future read of it.
- Letting the persistent file grow unbounded — fold or archive old entries once it passes ~5, or the next chat wastes tokens skimming stale ones to find what's current.
- Writing the persistent file only when a handoff is requested, never at session close — loses everything from a session that ended without one.
