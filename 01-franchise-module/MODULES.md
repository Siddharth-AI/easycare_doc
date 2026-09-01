# Project Modules — Phase 1

> Teen module: **Shop System**, **Product**, **Commission**.
> Language simple rakhi hai (client non-technical). Har point example ke saath.

---

# MODULE 1 — SHOP / FRANCHISE SYSTEM

Poora distribution structure — warehouse se lekar user tak.

## 1.1 Hierarchy (Levels)

```
LEVEL 0    WAREHOUSE (WH)              → main stock source
              │
LEVEL 0.5   MINI-WAREHOUSE (optional)  → kahi-kahi, ek ya zyada
              │
LEVEL 1     SHOP / FRANCHISE           → S1, S2, S3 ...
              ├──────────────┐
LEVEL 2   SUB-FRANCHISE   LEVEL 3   SUB-FRANCHISE
          → S3.1                    → S3.2
```

> **⚠️ Ye structure badla hai.** Pehle chain seedhi thi — L1 → L2 → L3 (L3, L2 ke neeche).
> **Ab L2 aur L3 dono seedha L1 ke neeche hain** — aapas me bhai-bhai, ek doosre ke neeche nahi.

**Rules:**
1. Top pe ek Warehouse (WH) — yahi se saara stock start hota.
2. Mini-Warehouse **optional** — kuch jagah hoga, kuch jagah nahi. Ek se zyada ho sakte. WH ke theek neeche baithta hai.
3. Neeche shops/franchises — Level 1, aur L1 ke neeche Level 2 aur Level 3.
4. **L2 aur L3 dono ka parent = L1.** Jaise: S3.1 (L2) → S3 ke under, S3.2 (L3) → **bhi S3 ke under** (S3.1 ke nahi).
5. Abhi **maximum 3 level** tak. Future me badha sakte (system me option rakhenge).
6. **Levels sparse ho sakti** — zaroori nahi har L1 ke neeche dono ho:
   - Kisi ke sirf L1 hai (S2 — neeche kuch nahi).
   - Kisi ke L1 + L2 (S1 → S1.1).
   - Kisi ke L1 + L3 (S4 → S4.2).
   - Kisi ke L1 + dono (S3 → S3.1 aur S3.2).
7. **Stock lene ka flow same hai** — WH, MWH, L1, L2, L3 sab apne upar wale se maal le sakte hain, jaise pehle lete the.

## 1.2 Pincode Mapping (Service Area)

Har shop kuch pincode "serve" karti. Mapping **neeche se upar** chadti hai.

**Example (S3 ki chain):**

| Shop | Apna pincode | Neeche se mila | Total serve karega |
|------|-------------|----------------|--------------------|
| S3.2 (L3) | p1, p2, p3 | — | p1, p2, p3 |
| S3.1 (L2) | p4 | — (L3 iske neeche nahi) | sirf p4 |
| S3 (L1)   | p5, p6 | p1,p2,p3 (L3 se) + p4 (L2 se) | p1 – p6 |

**Rule:** Child ke saare pincode automatically parent ke saath bhi map ho jaate. Upar wali shop apne + neeche walon ke saare pincode serve karti.

> **⚠️ Badla hua:** pehle L2 ko L3 ke pincode bhi rollup me milte the (kyunki L3 uske neeche tha). **Ab nahi** — L2 aur L3 alag branch hain, isliye L2 sirf apne pincode serve karta hai. Dono ka rollup **seedha L1 me** jaata hai.

## 1.3 User

1. Har user ka ek pincode hota.
2. User ka pincode jis shop me map hai, wahi uski **"home" shop**.
3. Ek pincode multiple level pe map ho sakta → user un sabse khareed sakta.

**Examples:**
- **U1** ka pincode **p6** → sirf **S3** serve karti p6 ko → U1 sirf S3 se le sakta.
- **U2** ka pincode **p1** → p1 ko **S3.2 (L3)** aur **S3 (L1)** serve karti → U2 in dono se le sakta. **S3.1 (L2) nahi** — uske paas sirf p4 hai.

## 1.4 Buying Limit (IMPORTANT)

1. **Home shop se → UNLIMITED.** Apne pincode wali shop se jitna chahe khareede.
2. **Doosri shop se** (jo uske pincode ki nahi) → **monthly limit ₹2000**.
   - Matlab U1 (home S3) agar S1, S1.1, ya S2 se kharidna chahe → mahine me max ₹2000 tak.
3. Limit **rupay (spend)** pe hai, quantity pe nahi.
4. Limit **har mahine reset** — naye mahine fir ₹2000.

### Special Limit Override (admin)
- Kabhi kisi user ki limit badhani ho → admin special limit set karta.
- Example: admin ne **₹10,000** diya →
  - Ye **total cap** hai us mahine ka = normal ₹2000 **+ extra ₹8000**.
  - Matlab us mahine user doosri shops se ₹10,000 tak le sakta.
- **One-time, sirf us mahine ke liye.** Agle mahine wapas normal ₹2000.

### Limit Increase Request
- User khud limit badha nahi sakta.
- User **request** bhejta → **Admin** ko jaati → admin approve/reject karta.

---

# MODULE 2 — PRODUCT

## 2.1 Category Structure (nested)

```
Category
   └── Sub-category
          └── Sub-sub-category
                 └── Sub-sub-sub-category
```

1. Categories nested (4 level tak).
2. **Product kisi bhi level pe** lag sakta (zaroori nahi sabse neeche ho).
3. Har category ka ek **prefix** hota (category name se) — product code me lagega.

## 2.2 Product Information (fields)

| Field | Example | Note |
|-------|---------|------|
| Unique ID | 123 | system ki unique id |
| Name | Rakshak | product name |
| Category | Ag | kis category me |
| Category prefix + name | Ag-Rakshak123 | auto-generated code |
| QR Code | — | scan ke liye |
| Barcode | — | scan ke liye |
| Packaging type | Box / Bottle / Bag | |
| Unit | ml / kg / pc | |
| MRP | 1000 | printed price |
| Sell Price (MSP) | 800 | user ko isi me milega (cost included) |
| Company ID | — | |
| Device ID | — | |
| Description | — | |
| Crop (fasal) | multiple | ek product multiple crops ke liye |
| Images | multiple | |

## 2.3 Franchise Pricing (level-wise)

Ek hi product alag-alag level pe alag price me jaata:

| Kisko | Price (example) |
|-------|-------|
| Level 1 (shop) | ₹700 |
| Level 2 | ₹725 |
| Level 3 | ₹750 |
| User (MSP) | ₹800 |

> **⚠️ Ye numbers system me FIX nahi hain.** Har **product** ke apne level-price hote hain — Admin product master me MWH / L1 / L2 / L3 ka rate set karta hai, aur commission usi product ke rate se nikalta hai. Upar ke numbers sirf ek **example** hain.

> **Billing ke waqt cost price customize** ki ja sakti hai — bill usi cost pe banega. Har bill me kaunsi cost use hui, wo record me save rehti hai.

## 2.4 GST

- GST percentage product pe set (example: **10%**).
- Split: **CGST 5% + SGST 5%**.

## 2.5 Points (Rewards) — placeholder

User (U1, U2, U3) ke liye points fields:
- **PV** = 20
- **UV** = 10
- **NV** = 0 / null

> Note: reward/points ka pura logic baad me define hoga (abhi sirf fields rakhe hai).

---

# MODULE 3 — COMMISSION

## 3.1 Concept

- Product warehouse se neeche jata, har level apne price pe khareedta, upar wale margin pe nahi — **commission** har level ko milta jahan se aage gaya.
- Har level ka commission **pehle se set** hota.

## 3.2 Price ladder — ab do alag raaste

Nayi structure me L2 aur L3 dono L1 ke neeche hain, isliye **ek hi chain me teeno kabhi nahi aate**. Do possible raaste:

```
Raasta 1:  WH ──► Level 1 ──► Level 2 ──► USER
Raasta 2:  WH ──► Level 1 ──► Level 3 ──► USER
Raasta 3:  WH ──► Level 1 ──► USER          (seedha)
```

- Har raaste me **sirf do level** ko commission milta hai (L1 + L2, ya L1 + L3).
- Purana `WH → L1 → L2 → L3 → USER` wala full chain **ab exist nahi karta**.

## 3.3 Commission split — har level ko apna

**Rule:** L2 aur L3 ke beech **koi rishta nahi hai**. Baaki flow bilkul pehle jaisa — har level ko **apna set commission** milta hai.

| User ne khareeda | Kis-kis ko commission |
|---|---|
| Level 2 se | **L1** + **L2** — dono ko apna-apna |
| Level 3 se | **L1** + **L3** — dono ko apna-apna |
| Level 1 se (direct) | **L1** ko, aur baaki 50% Warehouse ko (50/50) |

Ek raaste ka level doosre raaste me aata hi nahi, isliye aapas me kuch bantta bhi nahi.

> **Commission ka number product master se aata hai** — Admin har product pe har level ka rate set karta hai. System me koi fix ladder hardcode nahi.

## 3.4 Missing Level — roll-up rule ❌ KHATAM

Purana rule tha: *"missing level ka commission next upar wale node ko roll-up ho jata."*

> **Ab ye rule poori tarah khatam hai.** L2 aur L3 alag branch hain — L2 ka na hona L3 ko affect hi nahi karta, aur ulta bhi. Beech me koi "gap" ban hi nahi sakta.

## 3.5 Mini-Warehouse commission

Agar Mini-WH chain me hai:
```
WH ──685──► MINI-WH ──700──► Level 1 ── ...
```
- WH se Mini-WH ko ₹685 ka pada.
- Mini-WH → L1 ko ₹700 me deta.
- **Mini-WH ka commission = 700 − 685 = ₹15.**
- Baaki neeche ki chain same chalti.

---

# INVOICE / BILLING (commission ke saath related)

Stock neeche jate waqt document:

| Hop | Document |
|-----|----------|
| WH → Level 1 | sirf **Stock Transfer** note (GST invoice nahi) |
| Level 1 → Level 2 | **Tax Invoice** — L1 shop ka naam + GST |
| Level 1 → Level 3 | **Tax Invoice** — L1 shop ka naam + GST |
| Level 2 → User | Retail invoice (L2 shop) |
| Level 3 → User | Retail invoice (L3 shop) |

**Rule:** WH→L1 internal transfer challan. Uske baad har hop pe **bechne wali shop ki GST invoice** banegi.

---

# Deferred (abhi nahi — baad me)
- Inventory / Stock management (detail)
- Rewards / Points logic (PV / UV / NV)
- Payment / Wallet / Settlement
