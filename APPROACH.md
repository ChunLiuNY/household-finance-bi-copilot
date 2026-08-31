# Approach & Architecture

Technical write-up for [BI Copilot for Household Finance](README.md) — a self-hosted BI stack for household finance, with Claude Sonnet powering a narrative insights layer and a conversational bot.

Source code and data are private; this document describes the design.

---

## System overview

```
┌─────────────────────────────────────────────┐
│                  Browser                    │
│   React + TypeScript (Vite)                 │
│   Dashboard · Transactions · Insights ✨    │
│   Budgets · Upload                          │
│         │  TanStack React Query             │
│         │  (caching, invalidation)          │
└─────────┼───────────────────────────────────┘
          │ REST API (JSON) · localhost
┌─────────┼───────────────────────────────────┐
│         ▼         FastAPI                   │
│   ┌─────────────────────────────────────┐   │
│   │            API Routers              │   │
│   │ /dashboard /transactions /upload    │   │
│   │ /insights /budgets /cash /accounts  │   │
│   └────────────────┬────────────────────┘   │
│                    │                        │
│   ┌────────────────▼────────────────────┐   │
│   │           Services Layer            │   │
│   │  Parsers │ Dedup │ Categorizer      │   │
│   │  Aggregation │ Insights ────────────┼───┼──▶ Anthropic
│   └────────────────┬────────────────────┘   │    Claude API
│                    │                        │
│   ┌────────────────▼────────────────────┐   │
│   │      SQLAlchemy ORM + SQLite        │   │
│   │  transactions │ merchant_rules      │   │
│   │  uploads │ accounts │ budgets       │   │
│   └────────────────▲────────────────────┘   │
│                    │ direct DB access       │
│   ┌────────────────┴────────────────────┐   │
│   │          Telegram bot               │───┼──▶ Anthropic
│   │  Q&A │ cash logging │ history       │   │    Claude API
│   └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
          ▲ Telegram Bot API (polling)
┌─────────┴───────────────────────────────────┐
│            Phone (Telegram app)             │
│   "how much did we spend on dining?"        │
│   "log $15 coffee"                          │
└─────────────────────────────────────────────┘
```

The bot runs as a separate process against the same database rather than through the REST API — no HTTP round-trip, and it keeps working regardless of whether the web app is open.

---

## 1. Ingestion

Statements arrive as quarterly exports. Every institution formats them differently: different column names, date formats, sign conventions, and — for two of them — no CSV export at all, only PDF statements.

```
User drops CSV / PDF
        │
        ▼
┌───────────────────┐
│  Format detection │  Inspects filename + content signature
└────────┬──────────┘  to identify the source format
         ▼
┌───────────────────┐  One parser per institution/format (10 total)
│  Parse & normalize│  · Column mapping and date parsing
│                   │  · Debit/credit direction normalization
│                   │  · Exclusion rules (card autopay, internal transfers)
│                   │  · Income detection (payroll, direct deposit)
└────────┬──────────┘
         ▼
┌───────────────────┐  SHA-256 over account | date | description | amount
│  Deduplication    │  Skips rows already present
└────────┬──────────┘
         ▼
┌───────────────────┐
│  Insert (pending) │  Categorization runs async — upload returns immediately
└───────────────────┘
```

**Design notes**

- **One parser per format, deliberately.** A single "smart" parser with branching would have been shorter and far more fragile. Isolated parsers mean an edge case in one institution's export can't corrupt another's.
- **PDF is a first-class path.** Two issuers offer no CSV export, so those statements are parsed from PDF with `pdfplumber` — including a fees section whose line format differs from the purchases section (no reference number), which silently dropped annual fees until it was handled explicitly.
- **Deduplication by content hash** makes re-imports safe. Exports overlap at date boundaries constantly; without this, every quarterly import would double-count the seam.
- **Exclusions matter more than they sound.** A credit card autopay from checking is not a second expense — the purchases are already captured on the card statement. Left in, it inflates spending by roughly the entire card balance every month.

---

## 2. Entity resolution: the actual hard problem

Bank descriptions are hostile to grouping. The same merchant appears under dozens of distinct strings, each carrying a unique transaction code, store address, or reference ID.

**Normalization pipeline**, applied before any matching:

```
"EXAMPLE MKTPL*7Y41KQ2Z9 123 COMMERCE ST AUSTIN TX USA"
        │
        ▼  Lowercase
        ▼  Strip trailing state / country suffixes
        ▼  Strip street addresses
        ▼  Strip *transaction codes
        ▼  Strip 10+ char alphanumeric reference codes
        ▼  Strip originator IDs and long digit runs
        ▼  Collapse whitespace
"example mktpl"
```

**Matching** then runs three levels against a merchant-rules table (~910 rules, seeded from years of manually maintained spreadsheets):

1. Exact match on the normalized key
2. Stored key contained in the incoming description → longest match wins
3. Incoming description contained in a stored key

**The feedback loop is the point.** When no rule matches, the transaction lands as Uncategorized and I fix it in the UI. That correction is upserted back into the rules table keyed by the normalized description — so the same merchant is classified automatically from then on, across every account and every person. The rule set grows with use.

```
Evolution of categorization:
  v1:  Rule match → Claude fallback     (accurate, but cost + latency on every upload)
  v2:  Rule match only, better normalization  (fast, free, self-improving)
```

Removing the AI fallback was the right call once normalization was strong enough to carry the match rate. Inference cost recurs forever; a normalization improvement is paid for once. See the README for the longer version of this argument.

---

## 3. Warehouse schema

```
transactions
├── id, upload_id, account_id, person
├── transaction_date, post_date, description
├── amount            — always stored positive
├── is_debit          — direction flag (expense vs. income/credit)
├── is_excluded       — hidden from all calculations
├── category, subcategory, category_source
├── dedup_hash        — SHA-256
└── updated_at

merchant_rules
├── description_normalized, description_raw
├── category, subcategory
└── source            — seeded vs. user-corrected

uploads
├── filename, account_id, file_hash
├── row_count, imported_count, skipped_count
└── status            — pending → categorizing → categorized | error

accounts   — account_id, institution, type, person, is_joint
budgets    — year, category, subcategory, amount
```

**Amounts are always positive**, with direction carried by `is_debit`. Mixing signed amounts with a direction flag is how you end up with double-negatives in aggregate queries; picking one convention and enforcing it at the parser boundary removed a whole class of bug.

**`category_source`** tracks provenance — seeded from the historical spreadsheet, or manually set. Manual always wins, and manual is what writes back into the rules table.

---

## 4. Semantic layer

Between the raw table and anything a person reads sits a business-rules layer that encodes domain knowledge a generic schema can't express on its own. This is the part no off-the-shelf app could give me, and the reason building it was worth it. A few examples of the *class* of problem it solves:

- **Transfer vs. spend classification.** Not every debit is consumption — money moving between accounts (savings, investments, shared pots) needs to be excluded from spend and tracked as its own category instead, without being silently dropped or double-counted against income.
- **Ambiguous transaction types.** Some transactions look identical on a statement but mean opposite things economically (e.g. a credit that's a refund vs. one that isn't) — telling them apart determines whether they should offset spend or not.
- **Multi-party reconciliation.** When expenses are paid from a mix of individual and shared sources, netting out who owes whom requires rules a bank statement has no way to express.
- **Derived affordability metrics.** Headline numbers like income and total spend aren't the ones that answer "can I afford this?" — that requires composing several intermediate figures into one derived metric.

All of it is deterministic Python over the warehouse. None of it is delegated to a model.

---

## 5. AI layer

Claude Sonnet (`claude-sonnet-4-6`) is used in exactly two places, both chosen because the task is *language*, not *computation*.

### Insights — one-shot narrative

```
Backend computes:
  ┌──────────────────────────────────────┐
  │  Monthly income per person           │
  │  YTD spending by category            │
  │  Budget vs. actual                   │
  │  Settlement balances                 │
  │  Investment contributions            │
  │  Guilt-free spending per person      │
  └──────────────────┬───────────────────┘
                     ▼
              claude-sonnet-4-6
                     ▼
  4-paragraph narrative — budget status, settlement,
  savings capacity, guilt-free spending
```

Cached in memory per year, regenerated on demand via a Refresh button. The model receives computed facts and produces explanation; it never derives a figure itself. That boundary is what makes the output safe to trust — a hallucinated adjective is survivable, a hallucinated balance is not.

### Bot — multi-turn Q&A

Every message rebuilds the system prompt from live aggregates (YTD by category and subcategory, income per person, monthly totals, current-month breakdown), so answers always reflect the current database. Nothing is cached; at household data volumes the query is trivially fast, and staleness would be worse than latency.

The last ten exchange pairs are kept per chat so follow-ups resolve against context. History lives in memory only — it's never persisted, so no financial conversation ends up on disk.

**Cash logging** takes a different path entirely: `log $15 coffee` is parsed with a regex, the category inferred from a keyword map, and the row written directly. No model call — it would add latency and failure modes to something a regex handles deterministically.

---

## 6. Security & privacy

The entire motivation for building this was not handing financial data to a third party, so the boundaries are explicit:

- **Runs on localhost.** No hosting, no cloud database, no bank credentials shared with any aggregator. Statements are files I download myself.
- **Only aggregates reach the API.** Category totals and monthly averages — never raw statements, transaction-level detail, or account numbers.
- **Secrets in `.env`**, gitignored alongside the database, raw statement files, and historical spreadsheets.
- **The bot is allowlisted** to specific chat IDs and refuses to start if the allowlist is empty. Any message from an unknown chat is silently ignored — a Telegram bot token is a public endpoint, and an unauthenticated bot over financial data would be the single largest hole in the system.

---

## Key design decisions

| Decision | Rationale |
|---|---|
| SQLite over Postgres | Single-user, no concurrency needs, zero ops. The data fits in memory. |
| Rules over AI for categorization | Started with a Claude fallback and removed it — normalization made it unnecessary, and inference cost recurs forever |
| Deterministic math, AI narration | A wrong adjective is survivable; a wrong balance is not |
| Content-hash deduplication | Quarterly exports overlap at the seams; makes re-import idempotent |
| All amounts positive + direction flag | One convention, enforced at the parser boundary, kills a class of sign bugs |
| One parser per format | Isolation over cleverness — an edge case can't spread |
| Background categorization | Upload returns immediately; classification runs async |
| Rules shared across all accounts | A correction learned once benefits every account and person |
| Bot reads the DB directly | No HTTP round-trip; works whether or not the web app is running |
| Bot context rebuilt per message | Answers always current; staleness is worse than a few ms of latency |
| Conversation history in memory only | Nothing sensitive persisted to disk |
