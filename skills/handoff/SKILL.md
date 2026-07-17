---
name: handoff
description: Update Obsidian memory at the end of a session. Auto-generates the Last Session handoff, updates Active Context, and refreshes the relevant project file(s). Use before ending any productive session.
argument-hint: "[optional notes]"
---

# Session Handoff Protocol

Save session progress to Obsidian memory so the next session starts exactly where this one left off.

## Step 1: Analyze the Session

Review the current conversation to identify:
- **Completed work** — features built, files created/modified, research done, proposals written
- **Files created** — new files added (with paths)
- **Files updated** — existing files modified (with paths)
- **Deferred items** — things discussed but not done (with reason: blocked, out of scope, waiting on X)
- **Next actions** — what should happen next session
- **Decisions made** — architectural choices, tool selections, approach decisions

If `$ARGUMENTS` is provided, incorporate those notes into the handoff.

Then run a semantic lookup to surface related vault notes that may need updating:
```
mcp__smart-connections__lookup(query: "<project or topic of this session>", limit: 5)
```
Use the results to catch any related project files, reference docs, or context notes beyond the primary project file. If the project isn't immediately clear from the conversation, use the lookup to identify it.

## Step 1 Output: JSON Changelog

After analyzing the session, produce this JSON before dispatching Haiku. Haiku renders the exact strings — no synthesis, no rewriting.

```json
{
  "session_date": "YYYY-MM-DD",
  "project": "ProjectName or null for cross-project sessions",
  "active_context": {
    "header": "## Active Context — ProjectName (updated YYYY-MM-DD)",
    "lines": ["exact line 1", "exact line 2", "exact line 3"]
  },
  "last_session": {
    "completed": ["exact bullet 1", "exact bullet 2"],
    "created": ["relative/path/to/file.md"],
    "updated": ["relative/path/to/file.md"],
    "deferred": ["exact deferred item"],
    "decisions": ["exact decision item"],
    "next_action": "exact next action sentence"
  },
  "project_file": {
    "name": "ProjectName",
    "append_section": "exact markdown block to append"
  },
  "cache_pattern": {
    "slug": "kebab-slug",
    "topic": "one-sentence description",
    "tags": ["tag1"],
    "projects": ["ProjectName"],
    "libs": ["lib-name"],
    "body": "exact markdown body",
    "provenance": {
      "external": false,
      "source": "session_id",
      "tool": "claude_synthesis|handoff|defuddle|webfetch|exa|firecrawl|user_paste",
      "content_hash": "sha256:..."
    }
  },
  "cache_decision": {
    "slug": "kebab-slug",
    "applies_to": ["ProjectA"],
    "stack": ["next-js"],
    "body": "exact markdown body"
  },
  "decisions_append": {
    "file": "Cache/Decisions/YYYY-MM-DD_slug.md",
    "content": "exact markdown to append"
  },
  "skip_steps": ["step3", "step4b", "step4c", "step4d"]
}
```

Omit null fields entirely. `skip_steps` lists which steps Haiku should skip.

**Provenance gating (memory poisoning defense, v5.2+):**
The `provenance` block on `cache_pattern` / `cache_research` is REQUIRED when content originated outside the vault (defuddle, WebFetch, exa, firecrawl, MCP server outputs, pasted text). Set `external: true` + fill all 4 sub-fields (`source`, `tool`, `content_hash`, `ingested_at`). For ordinary session-synthesis content (Claude wrote it from session reasoning), set `external: false` and the provenance block is optional.

**Routing (Phase 2 — shipped 2026-05-16):**
Use `mcp__infi-memory__save_cache` to write Cache/ or Quarantine/ entries — it AUTO-ROUTES based on `provenance.external`:
- `external: true` → `Claude Memory/Quarantine/<sub>/YYYY-MM-DD_slug.md` (TTL 30d, not auto-loaded, tagged `[QUARANTINED]` in search)
- `external: false` or absent → `Claude Memory/Cache/<sub>/YYYY-MM-DD_slug.md` (trusted tier)

The `validate_note` MCP tool blocks writes with `external: true` but missing provenance fields. To promote a Quarantine note to Cache after review, call `mcp__infi-memory__promote_from_quarantine({ path: "Quarantine/<sub>/<file>", note: "<why>" })`. See [[Cache/Decisions/2026-05-16_memory-poisoning-defenses]].

## Dispatch: Steps 2–6 via Haiku Subagent

Pass the JSON changelog to Haiku. Haiku is a **mechanical rendering agent** — it writes the exact strings from the JSON into Obsidian. It does not interpret, summarize, or improve content.

```
Agent(
  model="haiku",
  description="Session handoff – Obsidian writes and vault backup",
  prompt="You are a mechanical rendering agent. Parse the JSON changelog below and execute Steps 2–6 of the handoff protocol. Use the EXACT strings from the JSON — do not rewrite, summarize, or improve them. Render, don't interpret.\n\nJSON CHANGELOG:\n<paste JSON here>\n\nNow execute Steps 2–6 of the handoff skill exactly as written.\n\n⚠️ PATH RULE (critical): ALL Obsidian MCP paths MUST be vault-relative and start with `Claude Memory/`. This applies to BOTH tool formats:
- New tools (`obsidian_write_note`, `obsidian_get_note`, `obsidian_patch_note`): use `target: {"type": "path", "path": "Claude Memory/Cache/Patterns/foo.md"}`
- Old tools (`obsidian_update_note`): use `targetIdentifier: "Claude Memory/Cache/Patterns/foo.md"`
NEVER use just `Cache/Patterns/foo.md` or `Codebase/foo.md` — those create stray root-level folders outside Claude Memory/. NEVER pass absolute filesystem paths (`/Users/...`). Use absolute paths ONLY with the Read/Bash tools for reading files, NEVER as Obsidian targets.\n\n⚠️ VALIDATE BEFORE WRITE (Step 4f): Before each mcp__obsidian__obsidian_write_note or mcp__obsidian__obsidian_append_to_note call, run mcp__infi-memory__validate_note(content: '<full note content>', skip_links: true). If valid=false (errors present): fix all errors first, then re-validate. If warnings only: proceed and list warnings in Step 6 report. Exception: skip validation for Claude Memory Index.md edits (too dynamic).\n\n⚠️ MOC REGISTRATION RULE (critical): After writing any project file (`Claude Memory/<ProjectName>.md`) or session archive file (`Claude Memory/Last Session — *.md`, `Claude Memory/Session_*.md`), immediately check whether `[[ProjectName]]` (or `[[Last Session — DATE]]` etc.) appears in `Claude Memory/Claude Memory Index.md`. Search the file content for the wikilink. If MISSING: append it to the appropriate section — `## Project Registry` for project files, `## Session Archive` for session logs — using obsidian_patch_note with operation=append on that section heading. This prevents MOC gap health warnings on every session start.\n\n⚠️ CANONICAL FOLDER ALLOWLIST (critical — prevents structural sprawl): The ONLY locations you may write inside `Claude Memory/` are: the project file itself (`Claude Memory/<ProjectName>.md`), standalone session files (`Claude Memory/Last Session — <ProjectName> (date).md` or `Claude Memory/Session_<date>_<ProjectName>.md`, loose in the `Claude Memory/` root — NOT in a subfolder), `Claude Memory Index.md`, `Codebase/<Project>/`, `Cache/{Patterns,Decisions,Research}/`, and `Quarantine/`. NEVER create a new top-level folder such as `Active Context/`, `Sessions/`, `Last Session/`, `Project Memory/`, or `Projects/` — a 2026-07-07 cleanup deleted 41 files and 5 such rogue folders that had forked the canonical structure (see `Cache/Decisions/2026-07-07_vault-canonical-folder-allowlist.md`). If unsure where something belongs, it belongs in the existing project file, not a new folder.\n\n⚠️ SAME-DAY SESSION FILE RULE (critical — prevents archive bloat): Before creating a new standalone `Last Session — <ProjectName> (date).md` file, check whether one already exists for that exact project + date. If it does, APPEND a new subsection to that existing file (e.g. `## Round 2 — <topic>`) instead of creating a sibling like `(cont'd 2)` or `(topic name)`. One file per project per calendar day, no exceptions.\n\n⚠️ SESSION ARCHIVE CAP (critical): After adding a new link to `## Session Archive`, count existing links for that same project. If more than 2 exist, take the oldest excess link(s): fold their content into the project file's `## History (compressed)` section as one-liners (`- YYYY-MM-DD: summary`), delete the standalone file, and remove the link from Session Archive. Session Archive should never hold more than 2 links per project.\n\n⚠️ GIT RULE (critical): The ONLY git push you are permitted to run is the Obsidian vault backup in Step 5: cd \"/Users/gochuicod/Documents/2nd Brain\" && git add -A && git commit ... && git push. NEVER push any other repository. NEVER run git commands outside of /Users/gochuicod/Documents/2nd Brain. Do NOT touch shipx-next, infi-crm, xyz-ccp, or any repo under /Users/gochuicod/Documents/InfiGroup/ — pushing a project repo to any branch without explicit user instruction is an unauthorized destructive action.\n\nGit push after all writes. Report result."
)
```

The subagent has full tool access (Bash, Read, Write, Edit, MCP). Return its report directly to the user once complete.

---

## Step 2: Update the Claude Memory Index

Read the current Index:
```
/Users/gochuicod/Documents/2nd Brain/gochuicod/Claude Memory/Claude Memory Index.md
```

Update TWO sections:

### Active Context (replace project section — multi-instance safe)
Write 2-3 lines capturing:
- What's currently in focus / the immediate priority
- What's next after that
- Any blockers or waiting items

Header format:
- Specific project session: `## Active Context — <ProjectName> (updated YYYY-MM-DD)`
- Root/cross-project session: `## Active Context (updated YYYY-MM-DD)`

⚠️ **Multi-instance safety:** Do NOT overwrite the entire Active Context area. Other Claude instances may have written their own project sections. Surgical update only:
1. Read the full Index content
2. Build the new section text
3. Search for `## Active Context — <ProjectName>` … next `## ` block → replace that block only
4. If not found: insert the new section before the `## Last Session` block
5. Write the full file back

### Last Session (replace project section — multi-instance safe)
Write a structured handoff:
```markdown
## Last Session — <ProjectName> (YYYY-MM-DD)
- Completed: [bullet list of what was done]
- Created: [new files with relative paths]
- Updated: [modified files with relative paths]
- Deferred: [what wasn't done and why]
- Decisions: [any notable decisions made]
- Next action: [what the next session should start with]
```

For root/cross-project sessions use `## Last Session (YYYY-MM-DD)` (no project suffix).

⚠️ Same surgical rule: only replace the matching project's Last Session section, preserve all others.

## Step 3: Update the Project File

If work was done on a specific project, update that project's Obsidian memory file:
```
/Users/gochuicod/Documents/2nd Brain/gochuicod/Claude Memory/<ProjectName>.md
```

**Before writing, run a staleness check:**
1. Read the file's `updated` frontmatter date
2. Count the file's total lines
3. Apply these rules:
   - If `updated` date is >30 days ago OR file is >80 lines: consolidate entries older than 60 days into a `## History (compressed)` section **before** appending new content
   - History entry format: `- YYYY-MM-DD: [1-sentence summary of what was done that session]`
   - Keep the last 2 sessions' entries in full detail; compress everything older into one-liners
   - Remove any known issues/bugs that were fixed this session
4. Record any stale files in the Step 6 report (see format below)

Follow the save rules:
- Save WHAT and WHY, skip HOW (unless surprising)
- One line per feature/change
- **Always** set the `updated` date in frontmatter to today's date (YYYY-MM-DD). This is critical for the tiered staleness system — without it, the project file will be flagged as stale on every handoff.
- If wiring changed, update the Wiring (compact) section

**Note on authoring:** Follow Obsidian Flavored Markdown conventions — use `[[wikilinks]]` for internal vault links (not plain markdown links), and `> [!note]` callouts for important context blocks. The obsidian-markdown skill is available if you need reference on syntax.

**Tool call shape for overwriting Obsidian files** — `wholeFileMode` is required or the call will fail:
```
mcp__obsidian__obsidian_update_note(
  targetType: "filePath",
  targetIdentifier: "Claude Memory/<file>.md",
  modificationType: "wholeFile",
  wholeFileMode: "overwrite",        ← required, or call errors with "wholeFileMode: Required"
  overwriteIfExists: true,
  content: "..."
)
```

## Step 3b: MOC Registration

After writing (or updating) any file in `Claude Memory/`:

1. For **project files** (`Claude Memory/<ProjectName>.md`): read `Claude Memory/Claude Memory Index.md`, search for `[[ProjectName]]` in the `## Project Registry` section. If missing, use `obsidian_patch_note` (operation=append, section heading=`Project Registry`) to append ` · [[ProjectName]]`.

2. For **session archive files** (`Claude Memory/Last Session — *.md`, `Claude Memory/Session_*.md`): same check against `## Session Archive` section.

3. Skip if the wikilink already exists — no duplicates.

This step takes < 5 seconds and prevents all MOC gap warnings.

## Step 4: Update Claude Code Memory (if needed)

If any feedback, user preferences, or reference info was learned this session, update the relevant file in:
```
~/.claude/projects/-Users-gochuicod-Documents-InfiGroup/memory/
```

Only update if something genuinely new was learned — don't duplicate what's already there.

## Step 4b: Surface Cacheable Patterns

Scan the session for non-obvious solutions worth caching. Promote to `Claude Memory/Cache/Patterns/` if ANY of:
- Solution took multiple attempts or was counterintuitive
- Pattern is a library/tool/API quirk (bot detection, version gotcha, obscure flag, workaround for a known bug)
- Reusable across projects (not tied to one codebase's specifics)

**Do NOT cache:** session-specific implementation details (those go in the project file), easily re-derivable info, one-off bug fixes unique to a single codebase.

**Format:** `Claude Memory/Cache/Patterns/YYYY-MM-DD_short-kebab-slug.md` with frontmatter `tags: [cache, pattern, <lib>]`, `created:`, `topic:`, `projects: [...]`, `libs: [...]`. Body: **Problem**, **Solution**, **Why it works**, minimal code shape, **Applies when**.

For expensive external research done this session (exa/firecrawl/deep web ops, >3k tokens of synthesis), promote via `mcp__infi-memory__save_cache({ kind: "research", provenance: { external: true, source, tool, content_hash, ingested_at } })`. This auto-routes to `Claude Memory/Quarantine/Research/` (TTL 30d). After reviewing the entry, promote to trusted Cache/ with `promote_from_quarantine`. For session-synthesized notes (no external source), pass `provenance.external: false` or omit `provenance` — routes to `Cache/Research/` directly.

**Dedup check (mandatory before writing):** Call `mcp__smart-connections__lookup(query=<topic>, limit=3)`. If any existing Cache/ entry scores >0.75 to your topic, **update that entry** instead of creating a new one — extend `projects:`, `libs:`, **Applies when**, and **Related** sections. Only create a new note if no close match exists.

**Contextual chunking (new notes only — do NOT backfill existing entries):**
Before writing a **new** `Cache/Patterns/` or `Cache/Research/` note, run a context lookup and prepend a `> [!context]` callout to the note body:

1. Call `mcp__smart-connections__lookup(query: "<note topic>", limit: 3)` — use the same query as the dedup check (results may already be in hand).
2. From the top-3 results, identify any that **extend, complement, or supersede** this new note (ignore unrelated hits).
3. Generate a short callout (50–100 tokens) naming those relationships. Use `[[wikilinks]]` to reference the related notes by their vault stem (no path prefix needed for top-level notes; include subfolder for Cache/ notes).
4. Prepend the callout **before** the ## Problem section (or equivalent first section) in the note body.

Callout format:
```markdown
> [!context] Related memory
> Extends [[existing-note-stem]] (same library, different scenario). Complements [[another-note-stem]] (broader pattern). See also [[third-note-stem]] for the original decision context.
```

If no related notes score >0.4 or none are meaningfully related, **omit the callout entirely** — do not generate empty or low-signal callouts.

## Step 4c: Promote to Decisions?

Was a directional architectural choice made this session that:
- Applies to **more than one project**, OR
- Encodes a "we chose X over Y" rationale that should survive project file compression

→ **Yes:** Write `Claude Memory/Cache/Decisions/YYYY-MM-DD_slug.md`

**Format:**
```yaml
---
type: decision
date: YYYY-MM-DD
applies_to: [ProjectA, ProjectB]   # or [all] or [all-ecommerce]
stack: [next-js, shopify, stripe]
tags: [cache, decision]
---
```
Body sections: **Decision**, **Context**, **Rationale**, **Alternatives**, **Supersedes**, **Applies when**

**Dedup check:** Run `mcp__smart-connections__lookup(query=<decision topic>, limit=3)` first. If an existing decision entry covers the same choice, update it instead.

→ **No:** Skip

## Step 4d: Update Codebase Knowledge?

Did any of the following change this session?
- React components added, removed, or significantly changed
- Database schema migrated (new table, column change, relationship change)
- New API routes added or existing ones removed/changed
- Integration wiring changed (new third-party service, middleware updated)
- Environment variables added or removed

→ **Yes:** Update relevant `Claude Memory/Codebase/[Project]/*.md` file(s)
  - Use the existing file format (YAML frontmatter + markdown table/sections)
  - **Always** update `updated:` frontmatter to today's date
  - If the `Codebase/[Project]/` directory doesn't exist yet, create it now

→ **No:** Skip

## Step 4e: Link Check Before Push

Run a brief link check to catch any broken links before committing to GitHub:

```
mcp__infi-memory__check_links({ mode: "brief" })
```

- If result has 0 broken links and 0 orphans: proceed silently.
- If issues found: include a `### Link Issues` section in the Step 6 report listing them.
- **Non-blocking** — proceed to Step 5 regardless of what's found.

## Step 4f: Validate Notes Before Write

**Pre-write validation is now embedded in the Haiku dispatch prompt** — Haiku calls `mcp__infi-memory__validate_note` before each vault write automatically.

This step documents what the validator checks (for reference):
- **Frontmatter schema**: `tags`, `created`, `updated` fields present
- **Secrets scan**: API keys, passwords, DB connection strings, JWT tokens
- **Wikilink resolution**: `[[links]]` resolve to known vault files
- **Length**: note > 80 lines triggers a warning

Validator is pure static analysis — no LLM call, < 5ms. Errors block the write; warnings are logged only.

## Step 4g: Sleep-Time Consolidation (v5.4)

Call the consolidator to apply mechanical lifecycle operations after writes:

```
mcp__infi-memory__consolidate_memory({ dry_run: false, summarize_after_days: 14 })
```

The tool does three mechanical phases — no LLM call by the tool itself:
- **Phase 1 (decay):** ECC `strength` decays per Ebbinghaus (half-life 180d).
- **Phase 2 (demote):** Hot→Warm→Cold when strength<0.1 AND last_used>28d. Updates frontmatter + moves entry in `MEMORY.md`.
- **Phase 3 (summarize plan):** Returns `stale_block_count` for `## Last Session — *` blocks in Claude Memory Index.md older than `summarize_after_days`.

If `phase3_summarize.stale_block_count > 0`: optionally dispatch a Haiku subagent to compress each stale block into a 1–2 sentence episodic summary and replace it inline in Index.md (Karpathy LLM Wiki v2 lifecycle: working → episodic → semantic → procedural). Cheap, in-session, no async API.

**Non-blocking** — proceed to Step 5 regardless.

## Step 5: Backup to GitHub

After all Obsidian files are updated, commit and push the vault to GitHub:

```bash
cd "/Users/gochuicod/Documents/2nd Brain"
git add -A
git commit -m "chore: session handoff $(date +%Y-%m-%d)"
git push
```

If the push fails, switch to HTTPS first:
```bash
git remote set-url origin https://github.com/gochuicod/2nd-Brain.git
git push
```

Report success or any errors to the user.

## Step 6: Report to User

Provide a concise summary of what was saved:
```
Session handoff saved.

Active Context updated:
- [2-3 lines]

Last Session recorded:
- Completed: [count] items
- Deferred: [count] items
- Next: [next action]

Project file updated: [ProjectName].md
⚠️ Stale files consolidated: [list files that were >30 days or >80 lines, or "none"]
```

## Rules
- Keep the handoff concise — future Claude reads this at session start
- Use absolute dates (2026-03-19), never relative ("today", "yesterday")
- Don't save low-level details (CSS tweaks, line-by-line fixes)
- Don't save things already captured in git history
- If no meaningful work was done, say so — don't fabricate a handoff
- Token budget: project files should stay under ~80 lines (~1000 tokens). If a handoff would push the file over, consolidate before writing.
