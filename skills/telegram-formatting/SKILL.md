---
name: telegram-formatting
description: Formats all Telegram bot replies to be short, scannable, and properly styled with MarkdownV2. Use when replying to Telegram messages via the Telegram MCP plugin — ensures bold, emojis, bullet points, and correct special character escaping.
---

# Telegram Formatting — Claude Code Reply Style

Ensures every Telegram reply is **short, concise, and mobile-friendly** using MarkdownV2.

## When to Use

Apply this skill whenever replying via `mcp__plugin_telegram_telegram__reply`. Every reply must use `format: "markdownv2"`.

## Rules

| Rule | Requirement |
|------|-------------|
| **Length** | Under ~800 chars when possible |
| **Format** | Always set `format: "markdownv2"` |
| **Structure** | Emojis as section headers + bullet points |
| **Emphasis** | Bold key terms and outcomes |
| **Tone** | Direct, no filler, no preamble |

## MarkdownV2 Syntax

| Entity | Syntax |
|--------|--------|
| Bold | `*bold text*` |
| Italic | `_italic text_` |
| Underline | `__underline__` |
| Strikethrough | `~strikethrough~` |
| Spoiler | `\|\|spoiler\|\|` |
| Inline code | `` `code` `` |
| Pre block | ` ```pre``` ` |
| Link | `[text](URL)` |
| Bold italic | `*_bold italic_*` |

## Escaping (CRITICAL)

These characters **MUST** be escaped with `\` when they appear **outside** formatting entities:

```
_ * [ ] ( ) ~ ` > # + - = | { } . !
```

### Escaping by context

| Context | What to escape |
|---------|---------------|
| Plain text | All 18 special chars listed above |
| Inside `(...)` of links | `)` and `\` only |
| Inside code/pre | `` ` `` and `\` only |

### Common gotchas

- `Done!` must be `Done\!`
- `file.txt` must be `file\.txt`
- `A - B` must be `A \- B`
- `key=val` must be `key\=val`
- `(test)` must be `\(test\)`

## Message Template

```
✅ *Done\!* Brief outcome summary

📋 *What changed:*
• Item one
• Item two

⚡ *Next:* What happens now
```

## Anti-Patterns

| Don't | Do |
|-------|-----|
| Wall of text | Bullets with emoji headers |
| Repeat what user said | Jump to the answer |
| Plain text format | Always MarkdownV2 |
| Unescaped special chars | Escape every `.` `!` `-` `(` `)` etc. |
| Long explanations | Max 3-5 bullet points per section |
