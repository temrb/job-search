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
site:https://www.linkedin.com/jobs/view/* "Full Time" ("jobs in" "{LOCATION}") [intitle:"{TARGET}"] -"No longer accepting applications" after:{YYYY-MM-DD} ["message the job poster"] {NEGATIONS}
```

- `[]` = conditional, see §4.
- `{LOCATION}`, `{TARGET}`, `{YYYY-MM-DD}`, `{NEGATIONS}` are computed per §4–§5.

Full example — target `support engineer`, location `New York, NY`, range `last week` computed from `2026-09-08` → `after:2026-09-01`:

```text
site:https://www.linkedin.com/jobs/view/* "Full Time" ("jobs in" "New York, NY") intitle:"support engineer" -"No longer accepting applications" after:2026-09-01 -intitle:"intern" -intitle:"internship" -intitle:"lead" -intitle:"senior" -intitle:"sr" -intitle:"staff" -intitle:"head" -intitle:"principal" -intitle:"group" -intitle:"director" -intitle:"chief" -intitle:"manager"
```

With `hiring=1`, `"message the job poster"` is inserted immediately after
`after:2026-09-01` and before the negations; with `hiring=0` the string
above is unchanged.

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
- `true` → emit exact phrase immediately after `after:`, before `{NEGATIONS}`:

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

## 5. Default Negations — Always On (with Dynamic Suppression)

Append in the background on every query, last — after `after:` and after
`["message the job poster"]`:

```text
-intitle:"intern" -intitle:"internship" -intitle:"lead" -intitle:"senior" -intitle:"sr" -intitle:"staff" -intitle:"head" -intitle:"principal" -intitle:"group" -intitle:"director" -intitle:"chief" -intitle:"manager"
```

### Dynamic suppression with alias groups

Negations are organized into order-preserving alias groups; emission order
matches the flat list above minus suppressed terms:

```text
[intern, internship] [lead] [senior, sr] [staff] [head] [principal] [group] [director] [chief] [manager]
```

Tokenize `{TARGET}` by lowercasing, stripping `"`, splitting on
`[^a-z0-9]+`, and dropping empties. Suppress a whole group if any title
token exactly equals any term in the group (word-token match, not
substring; no stemming needed). Suppression is bidirectional within a
group: either alias suppresses the whole group.

- `Product Manager` → tokens `product, manager` → drop `-intitle:"manager"`, keep the other 11 negations.
- `Senior Operations` → tokens `senior, operations` → drop `-intitle:"senior"` and `-intitle:"sr"` (10 remain).
- `Sr Analyst` → tokens `sr, analyst` → drop `senior` + `sr`, same as above (bidirectional alias).
- `Intern` or `Internship` → drop `-intitle:"intern"` and `-intitle:"internship"`.
- Substring traps do NOT suppress: `Leader` keeps `-intitle:"lead"`, `Headhunter` keeps `-intitle:"head"`, `Managerial` keeps `-intitle:"manager"`.
- Empty title → keep all 12 negations.
- `Support Engineer` → keep all 12 negations.

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
5. Add `"message the job poster"` if `hiring=1`.
6. Add negation list, applying the alias-group suppression in §5.
7. URL-encode the full string and append to `https://www.google.com/search?udm=14&q=`.
