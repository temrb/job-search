# LinkedIn Jobs via Google — Query Builder

Tune filters to build a shareable Google search for open, full-time LinkedIn job posts. All state lives in the page URL — no backend, no tracking, no storage.

Live site: https://temrb.github.io/job-search/

## Use

Open the live site (or `index.html` locally), change Date / Location / Title / Hiring filters → copy the **Google search link** or share the builder URL.

Example builder link:

```text
https://temrb.github.io/job-search/?range=7d&loc=New+York%2C+NY&title=support+engineer&hiring=0
```

## Files

- `index.html` — self-contained app (inline CSS + JS, zero deps). Served as `/` by GitHub Pages.
- `query.md` — query-building spec.
- `.nojekyll` — disables Jekyll processing on Pages.

## Updates

Edit `index.html` → commit → push to `main`. Pages redeploys automatically in ~30–60s.
