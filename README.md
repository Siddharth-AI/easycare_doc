# Project Deliverables

Client project ke saare deliverables — module-wise organized. (Blueprint config root me alag hai.)

> **⚠️ Client meetings (31-Aug + 01-Sep) ke baad bade changes hue hain.**
> Kya-kya badla: **[CHANGES_STRUCTURE.md](CHANGES_STRUCTURE.md)**
> Saare sawaalon ke jawab: **[QUESTIONS.md](QUESTIONS.md)**
>
> Sabse bade changes: L1 Special hataya · L2 aur L3 dono ab L1 ke neeche (siblings) ·
> L1 postpaid / L2-L3 prepaid ka alag flow · chain rule fix · point ab 4 tarah ke ·
> carry-forward/washout ka poora rule (strong/weak chalu, matching pehle aaj ka business).
>
> **Master doc ab project root me hai → [index.html](index.html)**

## 📁 00-documents ⭐ START HERE
High-level documentation — **client ke liye alag, developer ke liye alag**.

| File | Kya | Kiske liye |
|---|---|---|
| `../index.html` ⭐ | **Live master doc — ab project ROOT me hai** (v1.1). Poora system, 11 sections, sign-off + confirmed points | **Client** |
| `Apolium_System_Overview_CLIENT.docx/.pdf` | ⚠️ **Purana (v1.0)** — abhi regenerate nahi hua, `index.html` dekhiye | Client |
| `Apolium_Technical_Architecture_DEV.docx/.pdf` | ⚠️ **Purana** — DB schema, incentive algorithms, scaling, NFRs | **Internal / dev** |
| `README.md` | Dono docs ka index + locked architectural decisions | Internal |

## 📁 01-franchise-module
Warehouse → Mini-WH → **Level 1 → (Level 2 · Level 3 dono L1 ke neeche)** → User · pincode mapping · buying limit · commission engine.

| File | Kya | Kiske liye |
|---|---|---|
| `Franchise_Module_Documentation.docx` | Full colorful doc (23 diagrams, tables, cases) | **Client** |
| `FLOW_INTERACTIVE.html` | Step-by-step interactive walkthrough (animate) | Client / internal |
| `FLOWCHARTS.html` | Saare flowcharts ek jagah | Internal |
| `MODULES.md` | Module spec (text) | Internal |
| `build_doc.py` | Docx generator (rule badle → `python build_doc.py` re-run) | Internal |

## 📁 02-product-module
Product catalog · categories · pricing · GST · points · **Location tree** · stock/batch (previous→update→remaining).

| File | Kya | Kiske liye |
|---|---|---|
| `PRODUCT_INTERACTIVE.html` | Production DB schema walkthrough (11 tables, ER, stock flow) | **Internal / dev** |
| `PRODUCT_UI_CLIENT.html` | UI screen mockups + explanation (no technical cheez) | **Client** |

## 📁 03-billing-module
Wallet / stock / fund-request / commission flows. **L1 postpaid (credit limit) vs L2/L3 prepaid (wallet recharge)** — do bilkul alag flow.

| File | Kya | Kiske liye |
|---|---|---|
| `BILLING_INTERACTIVE.html` | Step-by-step walkthrough — tabs A/B/C (customer/franchise/full), **D (L2/L3 prepaid flow)**, **E (postpaid vs prepaid)**, Rules | **Client / internal** |

## 📁 04-user-module
Binary MLM tree — mother id, two-leg (Left/Right) binary, infinite depth. Sponsor ID vs Union/placement ID, spillover.

| File | Kya | Kiske liye |
|---|---|---|
| `USER_MODULE_INTERACTIVE.html` | Live tree builder (click se node jodo) + Sponsor-vs-Union concept + Rules | **Client / internal** |

## 📁 05-point-incentive-module
**Chaar point** (Self / Direct / Direct Team / Team) per bill, BPI daily left-right matching, self-PV capping levels, Startup Incentive (**ab time limit ke saath**), Mentor's Incentive (sponsor-tree), Achievers/Recognition (**Direct Team target ke saath**).

| File | Kya | Kiske liye |
|---|---|---|
| `POINT_INCENTIVE_INTERACTIVE.html` | Live purchase→point-flow animation on the tree + BPI day/week simulator + capping slider + mentor ring + achievers table + open-questions tab | **Client / internal** |

> Stan.store research is **not** part of this client project anymore —
> moved to `work/research/stan-store/`.

---

## Notes
- **Client-facing:** franchise docx · `PRODUCT_UI_CLIENT.html`.
- **Internal/dev:** `PRODUCT_INTERACTIVE.html` (full schema), `MODULES.md`, `build_doc.py`.
- HTML files → double-click, browser me offline khulti (kuch install nahi).
- Docx files → Word me open/edit.
- Blueprint (CLAUDE.md, agent_docs/, .claude/ etc.) repo **root** me alag hai — ye deliverables uska hissa nahi.
- Ye folder ab `work/clients/apolium-client-deliverables/` par hai (pehle root me `project-deliverables/` tha).
