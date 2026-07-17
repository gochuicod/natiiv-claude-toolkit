---
name: context
description: Load project context from Obsidian vault mid-session. Reads the project's Obsidian memory file on demand — gotchas, current state, active tickets, key decisions.
argument-hint: "[project-name] [slice?]"
---

# Project Context Loader

Load the Obsidian memory file for the current project, mid-session, on demand.

## Step 1 — Identify the project

Args format: `[project-name]` or `[project-name] [slice]`

Valid slices: `components`, `wiring`, `api`, `schema`, `env`, `all`

**If args contain two words** (e.g., `Arka wiring`):
- First word = project name
- Second word = slice
- Skip to **Step 2a** (Codebase slice retrieval)

**If args contain one word** (e.g., `Arka`) or no args:
- Use as project name (or detect from CWD — see table below)
- Continue to **Step 2** (standard project context)

If no args, detect from the current working directory:

| CWD contains | Project name |
|---|---|
| `shipx-next` | ShipX |
| `amilo-vn-next` | Amilo VN |
| `base_system` | Base System |
| `astro` | Astro |
| `gloria` | Gloria |
| `fyn` | Fyn |
| `arka` | Arka |
| `portfolio` | Portfolio |
| `lumenx` or `lumex` | LuMenX |
| `wisedrive` | Wisedrive |
| `r1vals` | R1VALS |

If CWD doesn't match any pattern, ask the user which project to load.

## Step 2a — Codebase slice retrieval (slice args only)

Read using `mcp__obsidian__obsidian_read_note`:
```
Claude Memory/Codebase/<ProjectName>/<slice>.md
```

If slice is `all`, read all files in `Claude Memory/Codebase/<ProjectName>/` (run 5 reads in parallel for components, wiring, api, schema, env — skip files that don't exist).

**If file not found:**
> "No codebase knowledge found for `[Project] / [slice]`. Seed it by running `/handoff` after your next session working in this project — Step 4d will prompt you to create it."

**If file found:** Present content with header:
> **Codebase slice loaded: [ProjectName] / [slice]**
> *(Updated: [updated date from frontmatter] — verify against live code if working in this area)*

Then say: "Ready to use this as context, or need another slice?"

Stop here — do NOT continue to Step 2 or Step 3.

## Step 2 — Read the Obsidian file

Read using `mcp__obsidian__obsidian_read_note`:
```
Claude Memory/<ProjectName>.md
```

## Step 3 — Surface the context

Present with a brief header:

> **Context loaded: [ProjectName]**
> *(Obsidian vault — may lag codebase by ~1 session. Verify stale entries against live files.)*

Show the full file content. Draw the user's attention to:
- **Gotchas** section first — most immediately useful for avoiding known pitfalls
- **Current State** + **Active Tickets** — where things stand
- **Known Issues** — things to work around

After presenting, say: "Anything specific you need from this context, or ready to dive in?"

Then check if codebase knowledge exists for this project by reading `Claude Memory/Codebase/<ProjectName>/` — if any slice files are present, append:

> **Codebase knowledge available:** `/context [ProjectName] components` · `/context [ProjectName] wiring` · `/context [ProjectName] api` · `/context [ProjectName] env`
> Use these for structural context before starting a coding task.
