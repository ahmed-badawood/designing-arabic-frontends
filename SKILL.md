---
name: designing-arabic-frontends
description: Use when building, styling, reviewing, or polishing frontend UI with Arabic or RTL content — components, pages, forms, charts, dashboards — including Arabic typography and fonts, Eastern/Western numerals, date and currency formatting, bidirectional (bidi) text, RTL layout in Tailwind/React, or adding RTL support to an existing interface.
---

# Designing Arabic Frontends

## Overview

An Arabic UI is not a Latin UI with translated strings. Direction, script, digits, and calendar all have Latin-first defaults that are silently wrong for Arabic. Sections are ordered by how often even RTL-aware developers miss them in testing.

## 1. Declare an Arabic font (the most-missed item)

Tailwind's default `font-sans` is Latin-first; Arabic falls to inconsistent OS fallbacks.

- App-level: load an Arabic-capable family — IBM Plex Sans Arabic, Tajawal, Cairo, Almarai, or Noto Sans Arabic:
  ```js
  // tailwind config: fontFamily.sans
  ['"IBM Plex Sans Arabic"', '"Noto Sans Arabic"', 'system-ui', 'sans-serif']
  ```
- Drop-in component: state the assumption in a comment ("assumes the app loads an Arabic font") — never silently rely on it.
- SVG doesn't inherit page CSS reliably: pass `fontFamily` explicitly to chart tick/label/tooltip styles (recharts `tick={{ fontFamily }}` etc.).
- Avoid weights 100–300 for Arabic at UI sizes; use 400–700.

## 2. Line-height: Tailwind defaults clip Arabic

Arabic ascenders, descenders, and diacritics need ~1.6+ line-height. Tailwind `text-sm` = 1.43, `text-xs` = 1.33, `leading-tight` = 1.25 — all too tight.

- Arabic body, labels, hints, errors: add `leading-relaxed` (1.625) or `leading-7`. (`leading-6` only clears 1.6 on `text-sm` and smaller — on `text-base` it equals the 1.5 default.)
- Display-size lines count too: the most-missed spot is the big `text-2xl` stat/amount line ending in an Arabic suffix («كم», «٪») — it keeps a sub-1.6 default leading unless you add one.
- Never `leading-tight`/`leading-none` on Arabic; clipping risk doubles with `truncate` (overflow-hidden).

## 3. Digits, dates, currency: one deliberate policy

Both numeral systems are correct Arabic — Eastern ٠١٢٣ and Western 0123 — and CLDR's default legitimately varies **by locale**: `ar-EG`/`ar-SA`/`ar-IQ` resolve Eastern digits, `ar-MA`/`ar-TN`/`ar-DZ` resolve Western. Don't impose one out of habit.

- **Default policy: the user's locale decides.** Pass the user's own locale (`navigator.language`, or the app's locale setting) to `Intl.NumberFormat`/`Intl.DateTimeFormat` and accept its digit choice. In an Arabic-only UI, prefer the app's locale setting when the browser locale isn't Arabic at all — otherwise dates render with non-Arabic month names. Pin a numbering system (`-u-nu-latn` or `-u-nu-arab`) only as a recorded, deliberate product-wide decision — centralized in one constant, never per-surface; per-user consistency is the requirement, cross-user uniformity is a product choice.
- **The non-negotiable rule is consistency:** ONE numeral system per user per view — including validation messages, hints, and inline examples (mixing «٣ أحرف» in error copy with Western digits in the input's example is a real observed failure). Route every number through one shared formatter; never hand-build digit strings. Before adding a formatter, check for an existing shared util and match it.
- **Calendar trap:** the `ar-*` **calendar** default varies by engine/ICU version (modern Node resolves Gregorian; older runtimes and some browsers resolve Hijri Umm al-Qura) — digit defaults don't vary; they're stable per locale. Pin the calendar you intend (`-u-ca-gregory`, or a Hijri calendar deliberately) — unless the user's tag already carries an explicit `-u-ca-` extension, which is a genuine user preference: honor it. Hardcoded month or date labels («يناير», a date string in copy) are a calendar choice too — state Gregorian-vs-Hijri in a comment just as you would justify a `-u-ca-` pin. `nu` and `ca` are independent extensions; setting one does not touch the other:
  ```ts
  const loc = new Intl.Locale(userLocale);                 // e.g. navigator.language
  const tag = loc.calendar ? loc.toString()                // explicit user preference — keep it
            : new Intl.Locale(userLocale, { calendar: 'gregory' }).toString(); // preserves any -u-nu-*
  const fmtDate = new Intl.DateTimeFormat(tag, { dateStyle: 'medium' });
  const fmtNum  = new Intl.NumberFormat(userLocale);       // digits: the locale decides
  ```
- **Currency: never hand-concatenate symbol + amount.** Currency symbols aren't language-bound — $, €, £ are all at home in Arabic UI. Use `style: 'currency'` and let Intl place the marker: Arabic locales render it **after** the amount («89 €», «٨٩٫٠٠ €») and prepend an RLM (U+200F) so the token embeds correctly in RTL text (see §4 — don't wrap that output in `dir="ltr"`).
- `tabular-nums` on numeric columns. Inputs: accept Eastern-digit typing and normalize BOTH ranges — Arabic-Indic ٠-٩ (U+0660–9, most Arabic keyboards) and Extended Arabic-Indic ۰-۹ (U+06F0–9, Persian/Urdu keyboards):
  ```ts
  const toLatinDigits = (s: string) => s
    .replace(/[٠-٩]/g, d => String('٠١٢٣٤٥٦٧٨٩'.indexOf(d)))
    .replace(/[۰-۹]/g, d => String('۰۱۲۳۴۵۶۷۸۹'.indexOf(d)));
  ```

## 4. Mixed-direction boundaries (the subtle layout breaker)

First, the base: establish direction once at the app root — `<html dir="rtl" lang="ar">`. `lang` matters as much as `dir`: it drives OS/browser Arabic font fallback and screen-reader voice selection. A standalone component re-declares `dir="rtl"` (ideally with `lang="ar"`) or states the assumption in a comment.

Logical properties resolve against **each element's own computed direction**, not the page's. Observed bug: in an RTL form, a `dir="ltr"` input got `pe-11` (= padding-RIGHT) while its overlaid eye-toggle button inherited RTL, so `end-0` = LEFT — icon on the opposite side, overlapping the typed text.

- Keep an input and its overlaid adornments (icons, toggles, prefixes) in the **same direction context** — or use a deliberate physical `left-*`/`right-*` pair at the boundary with a comment saying why.
- Force LTR islands where content is inherently LTR: email/phone/password inputs (`dir="ltr"`), a country-code prefix group like «+20» or «+966» (wrap the whole group in one `dir="ltr"` flex), raw LTR tokens — URLs, emails, code, hand-written numbers — embedded in Arabic sentences (`<span dir="ltr">`). **Exception:** `Intl`-formatted output for an Arabic locale already begins with its own RLM — a `dir="ltr"` wrapper fights the formatter's bidi mark; render it as-is.
- Put `dir="ltr"` on **inline** spans, not block elements — on a block it also flips which edge the text aligns to.
- Phone numbers and digit-only tokens: `dir="auto"`/`<bdi>` falls back to **LTR** when no strong-direction character is found, so `<bdi>+201001234567</bdi>` does render correctly. Still prefer explicit `dir="ltr"` — it states intent and stays correct if Arabic text later joins the same element — or an LRM (U+200E) before tokens like a phone prefix («+20») inside plain Arabic strings. (Get the mechanism right when explaining: the raw *unisolated* span is what scrambles; bdi does not "fail".)

## 5. Logical-first CSS — and the leaks to audit

Use logical utilities even in an always-RTL app (survives future locales, reads as intent): `ms-/me-` not `ml-/mr-`, `ps-/pe-` not `pl-/pr-`, `start-/end-` not `left-/right-`, `text-start/text-end`, `rounded-s/e`, `border-s/e`. Prefer `gap-*` over `space-x-*` (else `space-x-reverse`).

Audit the usual leaks even in clean code: asterisks/badges with `mr-1`, absolutely positioned adornments with `left-*`, inline `marginLeft` styles. Tailwind has no logical `translate-x` — a hover nudge toward "forward" is physical, so write both halves explicitly: `group-hover:translate-x-1 rtl:group-hover:-translate-x-1` (or a comment noting the component hardcodes RTL) — an unpaired physical nudge is the observed failure.

- **letter-spacing: never on Arabic** — it visibly breaks the connected script. Audit `tracking-*` on mixed digit+Arabic spans too: «250 كم» contains Arabic glyphs.
- **No italics on Arabic** (no italic tradition; browsers fake-slant it). Use weight/color for emphasis.

## 6. Icons: mirror by meaning

- Mirror in RTL: forward/back arrows and chevrons (forward = `ChevronLeft`, back = `ArrowRight`), send, reply, undo/redo, indent.
- Never mirror: clock, search, checkmarks, media playback, logos, anything containing Latin glyphs or digits.
- Standardize flips on `rtl:-scale-x-100`. (`rotate-180` happens to work for vertically symmetric glyphs like chevrons, but silently breaks asymmetric ones — Send, Reply, diagonal arrows — so use scaleX everywhere and don't cite rotation as broken *for chevrons*.)
- Diagonal trend arrows: `ArrowUpRight` implies an LTR timeline; in RTL the timeline-correct diagonals are `ArrowUpLeft`/`ArrowDownLeft`. But when the flipped diagonal would acquire a second meaning in your product's iconography (flow semantics — upload/download, send/receive — rather than trend), drop the diagonal entirely: use vertical-only indicators (`TrendingUp`/`TrendingDown`, ▲/▼) or sign + color. State which convention you chose in a comment.

## 7. Charts (recharts) in RTL

recharts tooltip/hover math breaks inside RTL containers. Correct pattern: wrap the chart in `dir="ltr"`, set `XAxis reversed` + `YAxis orientation="right"` so time reads right-to-left; a custom tooltip re-establishes `dir="rtl"` for its Arabic content.

- Tick labels mixing digits with Arabic («4 آلاف») inherit the wrapper's LTR base direction and render digit-on-the-left; prefix an RLM (U+200F) or set direction on the tick text.
- Arabic counted nouns: 3–10 take the plural — «4 آلاف», not «4 ألف».

## 8. Forms

- Phone with a country code (+20, +212, +966, …): accept the local 0-prefixed form, bare, `+<code>…`, and `00<code>…`; normalize Eastern-digit input (both ranges, §3); store E.164. **Test the bare `+<code>…` paste path** — an observed regex `/^\+?00?<code>/` silently rejected the canonical international format while its doc comment claimed to accept it. **Also test typing character-by-character, not only pasting** — an observed input handler stripped the `+` on the first keystroke and its local-length cap then locked typed international input as permanently invalid.
- Arabic validation copy: gender agreement («كلمة المرور مطلوبة» / «رقم الهاتف مطلوب»), correct hamza, and no dangling «و» when copy branches conditionally. Use MSA vocabulary in copy examples — regional dialect words date and localize the product.

## Quick reference

| Topic | Rule |
|---|---|
| Font | Arabic family declared or assumption stated; never bare default `font-sans` |
| Line-height | ≥1.6 for Arabic text; never `leading-tight` |
| Digits | The user's locale decides (ar-EG → ١٢٣٤, ar-MA → 1234); pin `-u-nu-*` only as a product-wide decision; ONE system per user everywhere |
| Calendar | `ar-*` default varies by engine; pin the calendar you intend unless the user's tag carries `-u-ca-` |
| Root | `<html dir="rtl" lang="ar">` — `lang` drives font fallback + screen-reader voice |
| Currency | `style: 'currency'` — Intl places the symbol (after the amount in Arabic locales); never hand-concatenate |
| Letter-spacing / italics | never on anything containing Arabic glyphs |
| Spacing | logical utilities (`ms/me/ps/pe/start/end`); `gap` over `space-x` |
| LTR islands | email/phone/password inputs, country-code groups, raw LTR tokens — inline `dir="ltr"`; NOT Intl Arabic-locale output (it carries its own RLM) |
| Icons | mirror by meaning, `rtl:-scale-x-100`; trend arrows follow the RTL timeline |
| Charts | `dir="ltr"` wrapper + `XAxis reversed`; RLM on mixed digit+Arabic labels; pass fontFamily into SVG |

## Common mistakes

| Belief | Reality |
|---|---|
| "The app surely loads an Arabic font" | The single most-missed item in testing. Declare it or state the assumption. |
| "I pinned the calendar, so Intl is handled" | `nu` and `ca` are independent: `'ar-EG-u-ca-gregory'` still emits Eastern digits. |
| "Logical utilities are always direction-safe" | They resolve per-element. Mixed-dir contexts put `pe-*` and `end-*` on opposite physical sides. |
| "`tracking-tight` only touches the numbers" | The same span renders «كم» — Arabic glyphs get letter-spaced. |
| "Tailwind's default leading is fine" | `text-sm` = 1.43; Arabic wants ~1.6+. Diacritics and descenders clip. |
| "My phone regex covers the country code" | Test the exact paste `+<code>XXXXXXXXX` AND typing it key-by-key; observed bugs in both paths. |
| "Numerals just need to look Arabic" | Consistency is the rule — Eastern in errors + Western in examples is the failure. |
| "Arabic UI means Eastern digits" (or "Western is cleaner") | Both are correct Arabic; CLDR varies by region (Egypt/Gulf Eastern, Maghreb Western). The user's locale decides unless the product deliberately pins. |
| "Every embedded amount needs an LTR island" | Intl's Arabic-locale output already starts with an RLM; a `dir="ltr"` wrapper fights it. |
| "`<bdi>` fails on digit-only tokens" | It falls back to LTR and renders fine; explicit `dir="ltr"` is preferred for intent, not because bdi breaks. |
| "Arabic locales default to Hijri dates" | Only `ar-SA` is even a candidate, and modern ICU resolves Gregorian there; the default *varies* by engine — that's why you pin the calendar. |
