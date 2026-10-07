# Fakeblock — Job-Posting Verifier

Built by **Abhiram Reddy "Ready" Kandadi** for **Trust in the Hiring Funnel Hackathon @ NYU**
(NYU MakerSpace, Brooklyn, Sep 19, 2026) — co-hosted by localhost:nyc, Integral Recruiting, and
NYU. Track B: trust going out — impersonation and candidate scams.

**Award: Best Pitch** (category award, not a placement).

**Demo video:** https://youtu.be/-O5mWXlSM4k

> This README covers the `job-posting-verifier/` component only — the half of the repo built by
> Abhiram. The repo's other component, `email-scanner/` (an Outlook add-in that scans received
> recruiting emails for scam signals), was built by teammate Shrikar Swami and isn't described
> here.

## The problem

Scammers post fake job listings impersonating real companies. A job seeker sees "Software
Engineer at Acme," applies, and gets scammed out of money or personal data — while Acme, the
real company, has no idea it's happening. The usual defense is guesswork: does the listing *look*
fake? Weird grammar, a sketchy contact email, an odd domain? That guesswork gets harder every
year as AI makes fakes look more convincing.

## The idea

Most companies already publish their real, current openings through an ATS (Greenhouse, Lever,
Ashby, etc.) — and that list is public. So instead of inspecting a posting for signs of
forgery, Fakeblock just checks the list: if a company isn't actually hiring for the role a
posting claims, the posting is fake. No guessing, no LLM judgment call — a direct lookup against
the source of truth.

The demo: two columns. Left is every posting on the internet claiming to be from a given
company (scraped from job boards and aggregators). Right is that company's real, live opening
list, pulled straight from its ATS. Each left-column posting gets checked against the right
column and comes back **verified**, **unverified**, or flagged as having **no matching
requisition** — a likely fake.

## How it works

1. **Fetch the real list.** `lib/ats/` has adapters for Greenhouse, Lever, and Ashby
   (`lib/ats/{greenhouse,lever,ashby}.ts`). `resolveCompany()` (`lib/ats/index.ts`) takes a
   company display name (e.g. "Coinbase Inc."), derives candidate board tokens
   (`coinbase`, `coinbase-inc`, ...), and tries each provider until one returns roles. A live
   fetch is cached in memory for 5 minutes; if every live call fails, it falls back to a
   committed JSON snapshot in `data/cache/` so the demo never goes blank.
2. **Load the claimed postings.** `data/claimed-postings.json` holds the left column: postings
   scraped from real job boards (Adzuna, Jobrapido, etc. — tagged `origin: "real"`) plus a small
   set of clearly labeled fabricated postings (`origin: "fabricated"`, source "Demo sample") used
   to guarantee at least one visible fake in the demo.
3. **Match.** `lib/match.ts` compares each claimed posting's title against every real role's
   title using the better of a character-bigram Dice score (forgives typos) and a token Dice
   score (forgives word reordering), after stripping seniority words ("senior", "II", ...) and
   stopwords. Thresholds are deliberately conservative — a posting is only called out as fake
   (`no_such_req`) when *nothing* in the company's real listing is even loosely title-similar:
   - `score ≥ 0.85` → **verified**
   - `0.55 ≤ score < 0.85` → **unverified** (a related role exists, but it's not a clean match)
   - `score < 0.55` → **no_such_req** (flagged as a likely fake)

   Location is checked too (`locationCompatible`) but never changes the verdict on its own —
   aggregators routinely scatter a single remote role across arbitrary cities, so title
   similarity alone decides status; location only breaks ties between equally-scored roles and
   shows as an informational note in the case file.
4. **Display.** `app/page.tsx` renders the two-column UI; postings that come back `no_such_req`
   open a **case file** (`components/CaseFile.tsx`) showing the closest real role for
   comparison, via `app/api/role/route.ts`.

Self-test (`scripts/selftest.mts`) runs every one of a company's own real roles back through the
matcher as if they were claimed postings — the run against 217 live Coinbase roles came back
0 false `no_such_req`s (i.e. it never wrongly accuses a real listing of being fake).

## Architecture

```
job-posting-verifier/
├── app/
│   ├── page.tsx              # two-column UI (claimed vs. real postings)
│   ├── api/verify/route.ts   # runs the matcher for the claimed postings
│   └── api/role/route.ts     # fetches a single role's content for the case file
├── components/
│   └── CaseFile.tsx          # detail view for a flagged (no_such_req) posting
├── lib/
│   ├── ats/
│   │   ├── greenhouse.ts     # Greenhouse Job Board API adapter
│   │   ├── lever.ts          # Lever adapter
│   │   ├── ashby.ts          # Ashby adapter
│   │   └── index.ts          # resolveCompany(): token guessing, provider fallback, caching
│   └── match.ts               # title/location similarity + verdict thresholds
├── data/
│   ├── claimed-postings.json # left column: real scraped + labeled fabricated postings
│   ├── lensa-review.json     # real postings held back for having unreliable reworded titles
│   └── cache/*.json          # committed live-fetch snapshots (offline/demo fallback)
└── scripts/
    ├── selftest.mts          # self-verification against a company's own real roles
    └── snapshot.mjs           # regenerates the data/cache/*.json snapshots
```

**Stack:** Next.js 16 (App Router) + React 19 + TypeScript + Tailwind CSS 4. No database — the
ATS APIs and the committed JSON files are the only data sources. No deploy; the hackathon demo
ran on localhost only.

## Running it

```bash
cd job-posting-verifier
npm install
npm run build && npm start
# open http://localhost:3000
```

(`npm run dev` also works for local iteration.) The demo company was **Coinbase** (Greenhouse,
217 live roles at demo time); Stripe and Airbnb snapshots are included as fallback companies if
Coinbase's live API is unreachable.

To re-run the self-test against live data:

```bash
npx tsx scripts/selftest.mts
```

## Known limitations

- Only Greenhouse, Lever, and Ashby are supported — companies on other ATS platforms (Workday,
  iCIMS, etc.) aren't covered.
- Matching is title-only; a posting with a wildly different but semantically equivalent title
  (e.g. "Software Engineer II" vs. "Backend SWE") can under-match.
- Aggregator-sourced postings (e.g. Lensa) sometimes reword titles enough to trip false
  `no_such_req`s — those were identified and moved to `data/lensa-review.json` rather than
  shown as flagged in the demo.
- `unverified` (amber) postings aren't clickable in the UI — only `no_such_req` (red) postings
  open a case file.

## Contributions

Abhiram Kandadi built the job-posting verifier (ATS adapters, title-matching, verdict logic, UI). Shrikar Swami built the email scanner. Best Pitch, Trust in the Hiring Funnel Hackathon, NYU, 19 September 2026.

## Upstream attribution

This repository is a fork of [ShrikarSwami/trust-in-the-hiring-funnel-hackathon-nyu](https://github.com/ShrikarSwami/trust-in-the-hiring-funnel-hackathon-nyu), the team's original hackathon repo. The full commit history, including Shrikar Swami's work on `email-scanner/`, is preserved unchanged. Only this root README differs from upstream.
