# LinkedIn Jobs via Google — Query Builder Spec

## 1. Purpose

Build a portable Google search URL that finds open, Full-Time LinkedIn job posts,
filtered by freshness, location, and title.

All state lives in URL params so a query is shareable / bookmarkable with no backend.

Base endpoint:

```text
https://www.google.com/search?udm=14&q={QUERY}
```

- `udm=14` = Web-only results.
- `{QUERY}` is the assembled job query below, URL-encoded when appended.

## 2. Invariant — Applied to Every Query

These clauses are always present and are not user-editable:

```text
site:https://www.linkedin.com/jobs/view/* "Full Time" -"No longer accepting applications"
```

- `site:https://www.linkedin.com/jobs/view/*` — only LinkedIn job detail pages.
- `"Full Time"` — exact phrase match.
- `-"No longer accepting applications"` — exclude closed posts.

## 3. Query Template

Assemble `q` in this exact order:

```text
site:https://www.linkedin.com/jobs/view/* "Full Time" ("jobs in" "{LOCATION}") [intitle:"{TARGET}"] -"No longer accepting applications" after:{YYYY-MM-DD} {NEGATIONS} ["message the job poster"]
```

- `[]` = conditional, see §4.
- `{LOCATION}`, `{TARGET}`, `{YYYY-MM-DD}`, `{NEGATIONS}` are computed per §4–§5.

Full example — target `support engineer`, location `New York, NY`, range `last week` computed from `2026-09-08` → `after:2026-09-01`:

```text
site:https://www.linkedin.com/jobs/view/* "Full Time" ("jobs in" "New York, NY") intitle:"support engineer" -"No longer accepting applications" after:2026-09-01 -intitle:"intern" -intitle:"internship" -intitle:"lead" -intitle:"senior" -intitle:"sr" -intitle:"staff" -intitle:"head" -intitle:"principal" -intitle:"group" -intitle:"director" -intitle:"chief" -intitle:"manager"
```

## 4. Editable Filters

### 4.1 Date — `after:{DATE}`

No hardcoded date. UI is a select with three relative options:

| Option | Param value | Calculation (at build time from current date) |
| --- | --- | --- |
| Last 3 days | `3d` | `today - 3 days` |
| Last week | `7d` | `today - 7 days` |
| Last 30 days | `30d` | `today - 30 days` |

Rules:

- Always emit exactly one `after:` clause, formatted `YYYY-MM-DD`.
- Example: today `2026-09-08` + `Last week` → `after:2026-09-01`.
- Default if param is missing/invalid: `7d`.
- Persist via URL param (see §6).

### 4.2 Location — `("jobs in" "{LOCATION}")`

UI: select with 4 defaults + free-text input that overrides the select.

Default options:

```text
United States
New York, NY
San Francisco, CA
Austin, TX
```

Rules:

- Always emit exactly one clause in the form `("jobs in" "{LOCATION}")`.
- Precedence: if custom input is non-empty (after trim), it wins over the select.
- Preserve commas and casing, always double-quote, e.g. `("jobs in" "New York, NY")`.
- Persist both select value and custom value via URL params (see §6).

### 4.3 Hiring Team Signal — Optional, Default OFF

UI: boolean toggle / checkbox, default `false`.

- `false` → emit nothing.
- `true` → append exact phrase at the end of the query:

```text
"message the job poster"
```

This biases toward posts that expose a poster / hiring team contact.

### 4.4 Title Targeting — Single Optional Keyword

UI: single text input, optional.

Rules:

- Empty → emit nothing.
- Non-empty → emit exactly one clause, trimmed and lowercased:

```text
intitle:"{keyword}"
```

- Example: `Support Engineer` → `intitle:"support engineer"`.
- Singular only. Never accept a list — one title per query to avoid over-constraining Google.

## 5. Default Negations — Always On (with 1 Smart Exception)

Append in the background on every query, after `after:`:

```text
-intitle:"intern" -intitle:"internship" -intitle:"lead" -intitle:"senior" -intitle:"sr" -intitle:"staff" -intitle:"head" -intitle:"principal" -intitle:"group" -intitle:"director" -intitle:"chief" -intitle:"manager"
```

### Smart `manager` rule

If `{TARGET}` contains the word `manager` as a substring (case-insensitive),
omit `-intitle:"manager"` from the negation list to avoid self-cancellation.

- `Product Manager` → drop `-intitle:"manager"`, keep the other 11 negations.
- `Support Engineer` → keep all 12 negations.
- Match on substring is sufficient; no stemming needed.

Apply the same suppression pattern if future target/negation pairs overlap.

## 6. State Persistence — URL Params

No `localStorage`. All builder state is encoded in the page's own URL query params
so links are portable, bookmarkable, and shareable.

Proposed param names:

```text
?range=3d|7d|30d &loc={selected} &custom={override} &hiring=0|1 &title={keyword}
```

| Param | Meaning | Default | Example |
| --- | --- | --- | --- |
| `range` | date range selector | `7d` | `?range=30d` |
| `loc` | location select value | `United States` | `?loc=Austin%2C%20TX` |
| `custom` | custom location override; wins if non-empty | empty | `?custom=Denver%2C%20CO` |
| `hiring` | hiring-team toggle | `0` | `?hiring=1` |
| `title` | target keyword, raw (lowercased at build) | empty | `?title=support%20engineer` |

Rules:

- On load: read params → initialize controls → build Google URL.
- On change: update params via `history.replaceState` (no reload) + rebuild output.
- Invalid `range` falls back to `7d`. Missing params fall back to defaults above.
- Builder params and the output Google `q` are separate: builder params drive the UI;
  the output is a fully-formed `https://www.google.com/search?udm=14&q=...` link.

Example builder link:

```text
/app?range=7d&loc=New+York%2C+NY&title=support+engineer&hiring=0
→ builds the Google URL in §3
```

## 7. Build Checklist

1. Start from invariant in §2.
2. Resolve location: `custom` if non-empty, else `loc`.
3. Add `intitle:"{title}"` if `title` is non-empty.
4. Add computed `after:` from `range` + today.
5. Add negation list, applying the `manager` rule in §5.
6. Append `"message the job poster"` if `hiring=1`.
7. URL-encode the full string and append to `https://www.google.com/search?udm=14&q=`.
