---
name: handing-off-context
description: Use when a conversation has grown long (heavy token usage, many full-file reads, repeated large context) and work needs to continue in a fresh chat without re-reading everything from scratch — produces a compact handoff prompt and enforces graph/grep-first lookups over full-file reads in the new chat.
---

# Handing Off Context

## Overview

A long-running session accumulates full-file reads, repeated tool output, and stale context. Past a point, continuing in the same chat costs more tokens per turn than starting fresh — but starting fresh naively means re-deriving everything the hard way (reading files end to end again). This skill produces a **compact handoff prompt** that lets a new chat resume at full speed, and sets the rule that the new chat must prefer indexed/targeted lookups (a code graph, `grep`, `graphify query`) over reading whole files.

## When to Use

Symptoms in the CURRENT (old) chat:
- Several full-file `Read` calls on large files (1000+ lines) already happened
- The conversation has been summarized/compacted more than once
- You notice yourself about to re-read a file you already read earlier just to "remind yourself"
- The user is running close to a usage/token limit and work isn't finished

Do NOT use for short conversations, or when the task is nearly done — finish first.

## What to Do in the OLD Chat

1. Tell the user plainly that context is getting heavy and a fresh chat will be more efficient.
2. Produce a **handoff prompt** (see template below) for the user to paste into the new chat.
3. Do not dump raw transcript, full file contents, or tool output into the prompt — summarize state, not history.

## Handoff Prompt Template

Keep it under ~300 words. Structure:

```
## Handoff: <short task name>

**Estado actual:** <1-3 sentences: what's done, what's in progress, what's confirmed vs pending>

**Próximo paso concreto:** <the single next action, not a list of options>

**Archivos/símbolos clave (NO leer completos — usar grep/graphify primero):**
- <file:line or symbol> — <why it matters>

**Herramientas a usar antes de leer archivos:**
- graphify: `graphify query "<question>" --graph <path-to-graph.json>`
- grep/ripgrep for exact symbols before opening a file

**Skills relevantes a activar:** <skill names, only if truly applicable — don't list generically>

**Restricciones activas:** <any hard rules still in force, e.g. "no marcar X completo sin confirmación en vivo del usuario">

**No ejecutar ni modificar nada todavía:** solo confirma que entendiste el estado y pregunta qué se quiere hacer antes de tocar código o correr comandos.
```

The last line is not optional boilerplate — always include it verbatim. Without it, the new chat's own "Auto Mode" bias (make the reasonable call and keep going instead of stopping to check) can read the handoff's "próximo paso concreto" as authorization to act immediately, before the user has actually said go.

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

- Handoff prompt that pastes large chunks of prior transcript — defeats the purpose, costs tokens in the new chat too.
- Vague "continue where we left off" with no concrete next step — forces the new chat to re-derive state by asking questions.
- Listing every skill available instead of the 1-2 actually relevant ones.
- Forgetting to name the exact graph path — the new chat then has to search for it.
