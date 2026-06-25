# designing-arabic-frontends

> **📦 Now part of the `arabic` plugin.** Easiest install:
> `/plugin marketplace add ahmed-badawood/claude-plugins` → `/plugin install arabic`
> (bundles this + `writing-eloquent-arabic`). This standalone repo is **archived/read-only**; the `npx skills add` command below still works but won't get updates.

مهارة لـ Claude Code تعلّم الوكلاء بناء واجهات عربية صحيحة — اتجاه، خطوط، أرقام، ونص ثنائي الاتجاه.

A [Claude Code](https://claude.com/claude-code) skill that teaches agents to build **correct Arabic-first frontends**: RTL layout, Arabic typography, Eastern/Western numerals, locale-aware date and currency formatting, bidirectional text, icon mirroring, and RTL-safe Tailwind.

Most published frontend skills are English-centric — they treat RTL as an edge-case test input, not a design target. This skill fills that gap.

## Why trust it

This skill was built with TDD for documentation — every rule targets a failure observed in real agent testing, and every factual claim was verified empirically:

| Phase | Runs | Critical | Major | Minor |
|---|---|---|---|---|
| Baseline (no skill) | 10 | 0 | **7** | 20 |
| With the skill | 10 | 0 | **0** | 15 |
| After corrections, re-test | 4 | 0 | **0** | 5 |

Ten realistic Arabic UI tasks (list views, dashboards with charts, signup forms, code review, Q&A) were run by agents **without** the skill and audited by expert graders against a 10-dimension Arabic/RTL rubric. The documented failures became the skill's content. The same tasks were re-run **with** the skill: all major failures disappeared. Verification graders executed `Intl` claims in Node and bidi claims in live Chromium — and caught two errors in the first draft, which are corrected in this version.

A second TDD cycle made the skill **domain- and market-neutral**: adversarial audits ran generic tasks (per-user-locale digits, EUR pricing) plus leak tests against it, and the prescriptions that failed were rewritten — digits now follow the user's own locale instead of a pinned system, and currency guidance is symbol-agnostic ($, €, £ all work; `Intl` places them). The re-test came back with zero major failures.

## What it covers (ordered by how often agents get it wrong)

1. **Arabic fonts** — the single most-missed item (7/10 baseline runs shipped Arabic on Latin-only stacks)
2. **Line-height** — Tailwind defaults clip Arabic ascenders/diacritics
3. **Digits, dates, currency** — both numeral systems are correct Arabic and the user's locale decides; the engine-varying calendar default; letting `Intl` place currency symbols (never hand-concatenating)
4. **Mixed-direction boundaries** — logical CSS properties resolve per-element; the observed eye-toggle-overlaps-password bug
5. **Logical-first Tailwind** — and the physical-utility leaks that survive even in clean code
6. **Icon mirroring by meaning** — including diagonal trend arrows vs. colliding flow semantics
7. **Charts (recharts) in RTL** — the `dir="ltr"` wrapper + reversed-axis pattern, bidi of mixed digit+Arabic tick labels
8. **Forms** — country-code phone validation (pasted *and* typed), normalizing Arabic-Indic ٠-٩ and Extended Arabic-Indic ۰-۹ input

## Install

```bash
npx skills add ahmed-badawood/designing-arabic-frontends
```

Works with Claude Code, Cursor, Codex, and any other agent supported by the [skills CLI](https://github.com/vercel-labs/skills) — installs into the current project; add `-g` to install globally. Or install manually:

```bash
git clone https://github.com/ahmed-badawood/designing-arabic-frontends.git ~/.claude/skills/designing-arabic-frontends
```

Claude Code discovers it automatically. It triggers on any task touching Arabic or RTL UI and composes with frontend skills like `frontend-design` and `impeccable`, layering the Arabic/RTL rules on top.

The skill follows the [Agent Skills specification](https://agentskills.io/specification), so it also works with other agents that support `SKILL.md` skills.

## License

[MIT](LICENSE)
