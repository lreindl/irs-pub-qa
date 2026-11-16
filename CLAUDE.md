# irs-pub-qa

A public TypeScript RAG app that answers questions about IRS publications with exact citations, plus an eval harness that proves it works. This is a portfolio project: the README's numbers are the product. Read `PLAN.md` for the full 8-week plan, reading list and milestones.

## Start here

1. Read `PLAN.md` end to end.
2. Work on **week 1** only (see "Week 1 scope" below). Don't build ahead of the current week.
3. Before writing code in any area marked **Lukus writes**, stop and leave a stub with a clear TODO instead.

## Goal

Ship by Jan 8, 2027:

- a RAG app over IRS publications where every answer cites the publication, tax year and section
- standard retrieval metrics (recall@k, MRR, nDCG@10)
- three code-based custom metrics and three validated LLM judges
- an experiment table comparing retrieval setups
- evals running in CI on every PR
- a deployed demo and a results write-up

## Who writes what

This split is the point of the project. Lukus must be able to explain every line of the core logic in an interview.

**Lukus writes by hand (Claude Code does not implement these):**

- Chunking: fixed-size, heading-aware and table-aware (`src/chunking/`)
- Retrieval: vector query, hybrid search with RRF, year filter, reranker wiring (`src/retrieval/`)
- Eval scoring: recall@k, MRR, nDCG@10, citation accuracy, tax-year correctness, numeric exactness (`src/eval/metrics/`)
- Judge prompts and pass/fail definitions (`src/eval/judges/`)
- Judge validation math: TPR, TNR, Cohen's kappa (`src/eval/validation/`)
- The golden set questions and his own labels (`evals/golden/`, `evals/labels/`)

In these areas Claude Code may: create the file with typed function signatures and a `// TODO(lukus)` body, write unit tests against the signature (tests he can make pass), review his code when asked, and explain concepts. It may not fill in the implementation unless he explicitly asks for that specific function.

**Claude Code handles:**

- Project scaffolding: Next.js, TypeScript config, lint, formatting, env handling
- Database: Postgres + pgvector schema and migrations, seed scripts
- Ingestion: scraping IRS.gov HTML, parsing into sections, storing rows, running embeddings through Ollama
- The experiment runner shell: loading a config, calling Lukus's functions, writing results JSON
- Tests: unit and integration tests, fixtures, test helpers
- Tracing and logging plumbing, Langfuse integration (week 7)
- CI: the GitHub Action that runs the 30-question eval and fails on a drop (week 8)
- The demo UI and deployment
- README tables and charts generated from results files

## Stack

- TypeScript (strict), Node 20+, pnpm
- Next.js (App Router) for the app and API routes
- Vercel AI SDK for generation and streaming
- Postgres with pgvector, hosted on Supabase (local Supabase or plain Postgres in Docker for dev)
- Drizzle ORM and migrations
- Ollama for local embeddings and drafts. A paid API is used mainly for judges. Keep API spend under about $50 for the whole project.
- Vitest for tests

## Corpus

- IRS publications: **Pub 17** (Your Federal Income Tax), **Pub 501** (dependents, standard deduction, filing information), **Pub 590-A** (IRA contributions), **Pub 970** (education benefits)
- Editions: **2025** (current) and **2024** (prior year). Index 2025 first in week 1, then add 2024 in week 5.
- Source: scrape the HTML editions from IRS.gov. There's no API. Prior-year editions are in the IRS prior-year archive, some only as PDF.
- Each stored section row carries: publication, tax year, chapter, heading, text, source URL.
- Tables in the publications matter (amounts by filing status). Keep each table row with its headers when parsing.
- Cache raw HTML and PDFs under `data/raw/` (gitignored) and fetch politely: one request at a time, a short delay, a descriptive User-Agent with no personal details.
- Possible week 8 extension: add more publications (for example Pub 526) to show the harness generalizes. The corpus is config, not code.

### Reuse and labeling

- IRS staff-written content isn't copyrighted and may be copied. Credit the IRS in the README and the demo footer.
- Don't use the IRS or Treasury seal or logo anywhere.
- Label the demo and README clearly: "Study project, not tax advice."

## Milestones

| Ships by | Milestone | What's public |
| --- | --- | --- |
| Nov 27 | 1. Baseline + retrieval metrics | Repo, golden set, recall@k / MRR / nDCG for the baseline |
| Dec 11 | 2. Better retrieval | Experiment table across 5 retrieval setups |
| Dec 24 | 3. Validated judges | 3 custom judges with agreement numbers, first blog post |
| Jan 8 | 4. CI + demo + write-up | Evals on every PR, live demo, results post |

Weekly detail is in `PLAN.md` under "Week by week".

## Week 1 scope (Nov 16)

Claude Code's part:

- Scaffold the Next.js + TypeScript + pnpm project with Vitest, ESLint and Prettier.
- Add Drizzle with a `sections` table (publication, tax_year, chapter, heading, text, source_url, embedding vector) and the pgvector extension.
- Write the ingestion script: fetch the 2025 HTML for the four publications, parse into sections, store rows, embed with Ollama.
- Add a minimal chat route with the AI SDK that answers with citations, using a retrieval function Lukus writes (stub it with a TODO).
- Log every query, the retrieved sections and the answer as a trace (JSONL under `traces/` is fine for now).
- README skeleton: one-line pitch, architecture sketch, "Study project, not tax advice", IRS credit, how to run.

Lukus's part: the chunking and the vector retrieval query.

## Repo layout

```
src/
  app/              Next.js routes and UI
  ingest/           IRS scraping, parsing, embedding (Claude Code)
  chunking/         Lukus writes
  retrieval/        Lukus writes
  generation/       prompt assembly and citation formatting
  eval/
    metrics/        Lukus writes
    judges/         Lukus writes
    validation/     Lukus writes
    runner/         experiment runner shell (Claude Code)
  db/               Drizzle schema and migrations
evals/
  golden/           golden set (JSON): question, expected sections, exact answer, tax year, tags
  labels/           Lukus's human labels for judge validation
experiments/
  configs/          one file per retrieval setup
  results/          metrics JSON per run (committed, they feed the README)
traces/             gitignored
data/raw/           gitignored
```

## Conventions

- Small PRs, one concern each. Conventional commit messages (`feat:`, `fix:`, `test:`, `chore:`).
- Every metric and judge has unit tests with hand-checked cases.
- Experiment results are reproducible from a config file plus one command (`pnpm exp <config>`).
- One command runs the evals: `pnpm eval`.
- Record cost and latency alongside every metric.
- No secrets in the repo. Use `.env.local` and keep `.env.example` current.
- Don't add a dependency for something the plan says Lukus builds by hand (for example, no Ragas-style scoring library before week 7, when it's added only for comparison).
- Prefer plain functions and explicit types over frameworks. LangChain is out of scope.
- Skip Evalite (deprecated). Use Langfuse for tracing in week 7.
