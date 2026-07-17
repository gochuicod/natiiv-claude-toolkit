# Natiiv Claude Toolkit

The Claude Code **skills** and **agents** behind the Natiiv / InfiGroup engineering workflow — the tooling companion to the *Founding Principles (Engineering Manual)*.

Everything here is **Natiiv-authored** and built for our canonical stack: **Next.js 16 · TypeScript strict · Tailwind + Shadcn · Drizzle · Neon/Supabase · next-intl · Zod · Arcjet · Vercel** (the `base_system` starter).

> Skills set the *approach* and are invoked before the work. Agents are dispatched to *carry it out*.

---

## Skills

| Skill | Covers |
|-------|--------|
| [`infigroup-design`](skills/infigroup-design/) | UI, layout, Figma / Josh design handoffs, token system, anti-pattern (Impeccable) laws, 1:1 design port |
| [`infigroup-backend`](skills/infigroup-backend/) | API routes (`withApi`), DB schema, auth, backend architecture, security, observability |
| [`infigroup-performance`](skills/infigroup-performance/) | Next.js perf, Core Web Vitals, SSG/ISR, accessibility (WCAG AA) |
| [`infigroup-i18n`](skills/infigroup-i18n/) | Translation, locale switching, next-intl `[locale]` routing |
| [`infigroup-seo`](skills/infigroup-seo/) | Sitemaps, robots, JSON-LD, schema.org, GEO |

## Agents

| Agent | Dispatch when |
|-------|---------------|
| [`infigroup-agents:design`](agents/design.md) | Design-to-code, Figma handoff, token system, UI audit |
| [`infigroup-agents:backend`](agents/backend.md) | API routes, DB schema, auth, background jobs, architecture |

---

## Install

**Skills** — drop each skill folder into your Claude Code skills directory:

```bash
cp -R skills/* ~/.claude/skills/
```

Then invoke with a slash command, e.g. `/infigroup-design` or `/context <ProjectName>`.

**Agents** — register the two agents as a local plugin/marketplace, or copy the `.md` files into your agents directory. Dispatch via `subagent_type: infigroup-agents:design` / `infigroup-agents:backend`.

---

## Third-party tooling (not vendored here)

Our workflow also leans on two public plugins. They are **not** copied into this repo — install them from source:

- **everything-claude-code** (`planner`, `architect`, `code-reviewer`, `security-reviewer`, `database-reviewer`, `build-error-resolver`, `tdd-guide`, `refactor-cleaner`, E2E + security-review skills) — https://github.com/affaan-m/everything-claude-code
- **superpowers** (`code review` skill, `superpowers:code-reviewer` agent) — https://github.com/obra/superpowers

---

## License

MIT — see [LICENSE](LICENSE).
