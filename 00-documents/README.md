# 00-documents — High-level documentation

Do document — ek **client ke liye**, ek **hamare (developer) liye**. Dono **Hinglish** me hain.

| File | Kiske liye | Kya hai |
|---|---|---|
| `Apolium_System_Overview_CLIENT.docx` / `.pdf` | **Client** | **22 pages.** Business language me poora system + worked examples. Har rule likha hai taaki client padh ke bata sake kahan galat samjha. |
| `Apolium_Technical_Architecture_DEV.docx` / `.pdf` | **Internal / dev** | **40 pages.** DB schema, incentive engine algorithms, monorepo structure, 5-lakh scaling, NFRs, testing, research references. |
| `RESEARCH-db-scale.md` | Reference | Web research (5,600 words) — MLM tree DB design, real citations. |

---

## Client document (15 pages)

Jaan-boojh ke chhota rakha — client ko 30-page doc nahi dena. Sirf wahi jo usko **verify** karna hai.

1. Ye document kaise padhein — colour box ka matlab
2. System ek nazar me — **franchise network vs member network** ka farq
3. Module 1–3 — franchise, price ladder, billing/wallet
4. Module 4 — binary tree, sponsor vs placement, chain rule
5. Module 5 — poora incentive plan (points, BPI, capping, startup, mentor, achievers)
6. **Aapko kya-kya milega** — 4 portal + mobile app, aur kaunsa Google pe dikhega
7. Live demo — interactive HTML files
8. **⚠️ Aapse confirm karna hai — 8 questions, tick-box ke saath**
9. Kaise banega — 6 phases
10. Sign-off page

**Section 8 sabse important.** Uske bina Phase 5 (incentives) finalize nahi ho sakta.

---

## Developer document (38 pages)

1. System context + load derivation (3.2M ledger rows/month)
2. Domain model — do tree (binary placement + sponsor DAG), extreme-chain placement
3. Data model — poora PostgreSQL schema + money/points precision
4. Point fan-out engine — outbox, idempotency, **batch aggregation**
5. BPI matching engine — exact algorithm, capping table, chunked close
6. Mentor / Startup / Achievers engines
7. NFRs — auditability, performance, security, ops, **3 PostgreSQL traps**
8. Testing — mandatory fixtures (client ke apne numbers = regression tests)
9. Open items + locked decisions
10. **Repo structure, apps & deployment** — monorepo, SEO isolation, future-proofing
11. Research basis & references

---

## Monorepo — client ka requirement

```
apolium/
├── apps/
│   ├── web/          → landing + member login    [PUBLIC · Google me dikhega]
│   ├── franchise/    → franchise portal          [PRIVATE · noindex]
│   ├── admin/        → superadmin panel          [PRIVATE · noindex + IP lock]
│   ├── api/          → backend
│   └── mobile/       → React Native (future)
└── packages/
    ├── core/         → ⭐ BUSINESS RULES (plan engine, pure functions)
    ├── types/        → API contract
    ├── sdk/          → typed API client
    ├── ui/           → design system
    └── config/       → eslint · tsconfig · tailwind
```

### `packages/core` — sabse important decision

Poora incentive plan (capping, BPI matching, washout, mentor %, achiever) **ek hi jagah**, pure functions me. Backend ka nightly close, admin ka simulator, member portal ka preview, mobile app — sab wahi function call karte hain.

Fayda: plan badla → ek file badli → sab jagah lag gaya. Aur simulator vs actual payout kabhi alag aaye, wo **bug hai** — dono literally same code chala rahe hain.

### SEO isolation — ek zaroori baat

`robots.txt` ka `Disallow` **indexing nahi rokta** — sirf crawling rokta hai. Google ko URL kahin aur linked mil gaya to wo phir bhi index kar lega.

**Asli fix:** `X-Robots-Tag: noindex, nofollow` HTTP header, server level pe. Aur private apps pe `Disallow` mat lagao — warna crawler `noindex` header padh hi nahi payega.

---

## Locked architectural decisions

- **Closure table + materialized path dono** — placement immutable hai, `upline_path` se zero-join ancestor read; closure table set-based batch jobs ke liye. ~2.4 GB total.
- **Async fan-out, batch-aggregated** — `GROUP BY ancestor` **pehle** likhne se. Write amplification 40× → 10×, root row batch me **ek baar** update. Yahi hot-row contention khatam karta hai.
- **Append-only ledger** — kabhi update nahi, correction = contra entry
- **Denormalised leg counters** — `fillfactor = 85`, aur counter column pe **koi index nahi**
- **Monthly partitioning** — par partitioned table pe UNIQUE global nahi hota; idempotency key alag non-partitioned table me
- **`BIGINT` paise / milli-PV / basis points** — `NUMERIC` nahi (exact par slow), float bilkul nahi. Split **largest-remainder method** se.
- **`pg_advisory_xact_lock` only** — PgBouncer transaction mode session-level advisory locks tod deta hai

### Ek line me

> 1–5 lakh members pe **data volume chhota hai** (~50 GB). System todega: (1) synchronous fan-out se tree ke top pe permanent lock contention, (2) non-restartable close jo retry pe double-pay kar de. Ye do fix karo — baaki ordinary PostgreSQL hai.

---

## Regenerate karna ho to

Scripts scratchpad me: `docxkit.py`, `build_client_doc.py`, `build_dev_doc.py`.
Content badlo → `python build_*.py` → Word me kholo → `Ctrl+A` phir `F9` (TOC) → PDF export.

---

## Yaad rakhne wali baat

Client doc **business rules ka source of truth** (*kya*). Dev doc **implementation ka** (*kaise*). Conflict ho to client doc jeetega, dev doc update karna padega.
