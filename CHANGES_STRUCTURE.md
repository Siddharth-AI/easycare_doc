# Change Log — Client Meetings ke baad (2026-08-31 + 2026-09-01)

> Ye document batata hai **kya-kya badla** aur **kaunsi file me badla**.
> Saare sawaalon ke jawab: [QUESTIONS.md](QUESTIONS.md)
>
> **Meeting #2 (01-Sep) me sab confirm ho gaya** — commission split, L1 postpaid, credit limit, achievers, startup window, cancel/return, payout, aur carry-forward ka poora rule.

---

## 1. L1 Special / koi bhi special level — HATA DIYA

**Client:** *"L1 special and koi bhi special nahi hoga, normal L1 L2 aese honge"*

| File | Kya hataya |
|---|---|
| `02-product-module/PRODUCT_INTERACTIVE.html` | `l1_special_price` field, `lv-l1s` CSS class, pricing description |
| `02-product-module/PRODUCT_UI_CLIENT.html` | "L1 Special ₹760" price card, explain-text |
| `01-franchise-module/build_doc.py` | Case C2 "special model ON" diagram (`d_cC2`), section, summary-table row |
| `01-franchise-module/FLOW_INTERACTIVE.html` | Case C2 step, "special rule" wording |
| `03-billing-module/BILLING_INTERACTIVE.html` | "L1-Special" role poora — fund-approval chain, minus-limit group, alag scenario card, rules-table row |

**Ab:** fund request approve karne wale sirf **L1 · MWH**. Koi special variant kahin nahi.

**Note:** "Admin special limit" (buying limit override) **alag cheez** hai — client ne bola chhod do, wo untouched hai.

---

## 2. Franchise structure — L2 aur L3 dono L1 ke neeche

**Client:** *"ab l1 ke under me hi l2 and l3 aaenge"*

```
PEHLE:                          AB:
WH                              WH
 └ MWH (optional)                └ MWH (optional)
    └ L1                            └ L1
       └ L2                            ├── L2
          └ L3                         └── L3
```

**Iska asar (3 jagah):**

1. **Stock flow** — same hai, koi change nahi. Sab apne upar wale se maal lete hain jaise pehle.
2. **Pincode rollup** — L2 ko ab L3 ke pincode **nahi** milte. Dono ka rollup **seedha L1** me jaata hai.
3. **Commission chain** — ek hi chain me L1+L2+L3 teeno **kabhi nahi** aate. Sirf do raaste: L1+L2 ya L1+L3. Har level ko **apna set commission** (section 11 dekho)

| File | Kya badla |
|---|---|
| `index.html` | Section 3.1 me naya hierarchy diagram, 3.3 price ladder ke raaste, overview cards |
| `01-franchise-module/FLOW_INTERACTIVE.html` | Node positions, saare edges (`S31→S311` ab `S3→S311`), pincode steps, commission steps |
| `01-franchise-module/FLOWCHARTS.html` | Naming table, hierarchy mermaid, sparse cases, rollup diagram + table, invoice flow |
| `01-franchise-module/MODULES.md` | Hierarchy ASCII, rules, pincode table, price ladder, commission section |
| `01-franchise-module/build_doc.py` | 6 diagrams redraw, text sections — **docx regenerate ho chuka** |
| `02-product-module/PRODUCT_INTERACTIVE.html` | Location table — S3.2 ka `parent_id` ab 4 (S3), pehle 5 (S3.1) tha |

**Naming badla:** `S3.1.1 (Level 3)` → `S3.2 (Level 3)` — kyunki ab wo S3 ka direct child hai.

---

## 3. Billing — L1 POSTPAID, L2/L3 PREPAID

**Client:** *"l1 ko limit thi bina paisa diye mall purchase kar skta hai ex 15lakh"* · *"l2, l3 me inko money daalni padegi"*

| | **L1 — POSTPAID** | **L2 / L3 — PREPAID** |
|---|---|---|
| Model | Maal pehle, **paisa baad me** | **Paisa pehle**, maal baad me |
| Limit | **Credit limit** — **per franchise** (Admin set karta), example ₹15,00,000 | Credit limit nahi, sirf **stock limit** |
| Wallet | Payment se limit khaali hoti hai | **Recharge** se bharta hai |
| Billing ka paisa | L1 **real money pehle deta hi nahi** — wallet minus/plus hota rehta hai | **Wallet me jama** — ye **den-dari** hai |
| Commission | Margin ka 50% | Margin ka 50% — den-dari me **adjust** |
| Upar paisa dena | **Wallet-to-wallet transfer** | **Wallet-to-wallet transfer** |

**Client ka example (dono flow alag hai, mix nahi hote):**
- ₹2,00,000 wallet me daala → recharge ho gaya
- Us ₹2 L me se ₹1,00,000 ka maal becha
- ₹50,000 ka commission bana → commission wallet me
- WH/MWH ko paisa dena ho → wallet-to-wallet transfer

| File | Kya badla |
|---|---|
| `index.html` | Section 3.4 poora rewrite — do model cards, prepaid example table, wallet-to-wallet rule, "chaar balance" table me "kis pe lagta hai" column |
| `03-billing-module/BILLING_INTERACTIVE.html` | Tab D = "L2/L3 Prepaid Flow" (3 naye steps), Tab E = "Postpaid vs Prepaid" (2 naye steps), Rules tab me comparison table |

---

## 4. Chain rule — left ka left, right ka right

**Client:** *"left ka left me and right ka right me, sponser niche ki chain me — na bich me na edhr udhr. jo simulator me diya hai vah dekho"*

- `04-user-module/USER_MODULE_INTERACTIVE.html` me **pehle se sahi tha** (extreme-chain drill) — koi change nahi
- **root ka `index.html` me GALAT tha** — "U1 se Right" wali chain `U1 → U3 → U7` dikhati thi, par **U7, U3 ka LEFT child** tha

**Fix:** tree me U8 (U3 ka right) aur U9 (U8 ka right) add kiye. Ab right-chain sach me right → right → right jaati hai. Captions bhi rewrite kiye.

---

## 5. Point — ab 4 tarah ke (pehle 3 the)

**Client:** *"self, direct, direct team(achiever me lagte hai), team point (left team, right team)"*

| # | Point | Kisko milta | Kahan use hota |
|---|---|---|---|
| 1 | Self | Buyer ko | Daily capping level |
| 2 | Direct | Sponsor ko | Startup Incentive |
| 3 | **Direct Team** *(naya)* | Poori direct team ka business | **Achievers qualification** |
| 4 | Team | Har ancestor ko — Left/Right | BPI matching |

| File | Kya badla |
|---|---|
| `index.html` | Section 5.1 "Teen tarah" → "Chaar tarah", naya row + callout |
| `05-point-incentive-module/POINT_INCENTIVE_INTERACTIVE.html` | Rules deck "3 type" → "4 type" |

---

## 6. Carry forward & washout — poora rule (strong/weak CHALU hai)

**Client (meeting #2):** *"strong and weak rule rhnega washout ke liye"* · *"Matching pehle aaj ka business ka and then carry add"*

> ⚠️ **Pehli meeting ke baad humne ye rule galat likh diya tha** — humne "strong/weak hata diya" aur "matching pehle purana carry kharch karti hai" likha tha. **Dono galat the.** Ab sir ne 4 example de kar poora rule clear kar diya hai.

**Sahi rule:**

1. Dono side ka **total = purana carry + aaj ka business**
2. Matching = min(totalL, totalR), 1,000 ke multiple me neeche round, daily cap tak
3. **Strong side** = jiska **total** bada · **Weak side** = jiska **total** chhota — *(sir ne confirm kiya: strong/weak **total se hi** decide hota hai, sirf aaj ke business se ya sirf carry se nahi)*
4. **Matching pehle AAJ ka business kharch karti hai**, uske baad purana carry — *(ye order sabse zaroori hai)*
5. **Purana carry jo bacha → hamesha aage.** Old carry kabhi wash nahi hota
6. **Aaj ke business ka jo bacha → wahi wash hota hai**
7. **Weak side pe washout nahi hua → strong side pe bhi nahi hoga**
8. Matching bani hi nahi → kuch wash nahi, dono side pura carry

**Sir ke chaaro example — live code se verify ho chuke:**

| # | Left (carry + aaj) | Right (carry + aaj) | Match | Result |
|---|---|---|---|---|
| 1 | 0 + 500 = 500 | 0 + 300 = 300 | 0 | Matching bani hi nahi → carry **500 / 300**, koi wash nahi |
| 2 | 300 + 800 = 1,100 | 500 + 1,000 = 1,500 | 1,000 | Dono ka aaj ka business match me laga → **koi wash nahi**. Carry **100 / 500** |
| 3 | 2,000 + 0 = 2,000 | 900 + 100 = 1,000 | 1,000 | Carry **1,000 / 0**. Koi wash nahi |
| 4 | 2,000 + 800 = 2,800 | 0 + 1,200 = 1,200 | 1,000 | Strong = Left. L ka aaj ka 800 pura laga → carry **1,800**. R ka aaj ka **200 wash** |

**Purani sheet se bhi match:**

| Din | Old L | Old R | Aaj R | Match | Carry L | Carry R | Wash R |
|---|---|---|---|---|---|---|---|
| Day 1 | 1,59,400 | 2,047 | 0 | 2,000 | **1,57,400** | **47** | 0 |
| Day 2 | 1,57,400 | 47 | 2,394 | 2,000 | 1,55,400 | 47 | **394** |

> Day-2 ka washout ab **394** aata hai — jo **aapki sheet me hi likha tha**. Pehle hum 441 keh rahe the (galat order ki wajah se). Ab sahi hai.

| File | Kya badla |
|---|---|
| `index.html` | `calcBPI()` rewrite (aaj-ka-business-first + strong/weak guard), rule callout, chaar-example table, verify callout |
| `05-point-incentive-module/POINT_INCENTIVE_INTERACTIVE.html` | `bpiCompute()` rewrite, BPI deck ke Step 3/4/5 rewrite, chaar-example step add, rules table, confirm table |

## 7. Price aur commission — FIX nahi, product-based

**Client:** *"pricing ka structure same, product me hi dalenge, commision etc sab pricing fix nahi hai vah to product base hi hai"*

- Har **product** pe Admin har level ka rate set karta hai (MWH / L1 / L2 / L3)
- Commission usi product ke rate se nikalta hai
- Documents me jo numbers hain (₹700 / ₹725 / ₹750 / ₹800) — wo ab साफ़ **"example"** likha hai, rule nahi

---

## 8. Billing ke waqt cost price customize

**Client:** *"inventory me billing krenge tab customize kar sake cost price ko, bill banega tab ki cost"*

- Inventory se bill banate waqt **cost price change ho sakti hai**
- Bill **usi customize ki hui cost pe** banega
- Product master ka cost sirf **default** hai
- Har bill line me **us waqt ki cost save** hoti hai (audit ke liye)

---

## 9. Startup Incentive pe TIME LIMIT

**Client:** *"startup incentive me control dena hai activation date se month ya day wise, iske baad nahi banega. ex 180 days"*

- Window **ID activation date** se shuru hoti hai
- Admin isko **mahino me ya dino me** set kar sakta hai
- Window ke andar milestone achieve nahi kiya → member **Startup Incentive se bahar**
- Ye limit **pehle milestone** pe lagti hai

*(Exact default value pending — QUESTIONS.md Q6)*

---

## 10. Achievers — Direct Team target

**Client:** *"pv left 1lakh right 1lakh, direct 5000, direct team 25k. yah monthly hai and yah 1 time hai qulify karne ke liye"*

| Lv | Rank | Left PV | Right PV | Direct/mahina | Direct Team |
|---|---|---|---|---|---|
| 1 | Silver Director | 1,00,000 | 1,00,000 | 5,000 | **25,000** |
| 2–5 | — | (pehle jaisa) | (pehle jaisa) | (pehle jaisa) | ❓ pending |

- Left PV / Right PV / Direct → **har mahine** maintain
- **Direct Team → sirf ek baar**, pehli baar qualify karne ke liye. Uske baad zaroorat nahi

---

## 11. Meeting #2 ke jawab — sab confirm

| # | Sawaal | Jawab |
|---|---|---|
| Q1 | Commission split? | L2 aur L3 me **koi rishta nahi**. Har level ko **apna set commission**. Sale L2 se → L1+L2 · sale L3 se → L1+L3. Number product master se |
| Q2 | Roll-up rule? | **Khatam** |
| Q3 | L1 postpaid ka billing paisa? | L1 **kabhi real money pehle deta hi nahi**. Limit hai, wallet minus/plus hota rehta hai — jaise pehle |
| Q4 | L1 ki credit limit? | **Per franchise** — Admin har L1 ke liye alag set karta hai |
| Q5 | Achievers Direct Team L2–L5? | Table me **pehle se hi hai** (L1–L5). Left/Right PV monthly, **Direct Team sirf ek baar** qualify karne ke liye |
| Q6 | Startup ki time window? | **Admin ke haath me** — jo chahe rakhe, baad me badal bhi sakta hai |
| Q7 | Bill cancel / stock return? | **Poora flow reverse** — stock, commission, wallet sab |
| Q8 | Payout? | Mahine me **1 baar** · **TDS 2%** |

| File | Kya badla |
|---|---|
| `index.html` | Q1–Q8 ke confirmed callouts, Achievers table (Direct Team column merge), open-questions list poori tarah confirmed |
| `01-franchise-module/FLOW_INTERACTIVE.html` | "❓ Open" wala commission step → confirmed rule step |
| `01-franchise-module/FLOWCHARTS.html` | Section 5.5 → confirmed commission rule |
| `01-franchise-module/MODULES.md` | Section 3.3 / 3.4 → confirmed rule + roll-up khatam |
| `01-franchise-module/build_doc.py` | Section 4.6 + summary table → confirmed. **docx regenerate ho chuka** |
| `05-point-incentive-module/POINT_INCENTIVE_INTERACTIVE.html` | Achievers table, Startup window (admin-configurable), open-questions tab → confirmed |

---

## ⚠️ Ek aur cheez — index.html ab ROOT me hai

Pehle ye `00-documents/index.html` par tha, ab project **root** me hai (`index.html`).
Uske andar ke demo links (`../01-franchise-module/...`) toot rahe the — **theek kar diye gaye** (`01-franchise-module/...`).

---

## Files jo change hui

```
index.html                                           ← master client doc (root me, v1.1)
01-franchise-module/build_doc.py                     ← docx regenerate ho chuka
01-franchise-module/Franchise_Module_Documentation.docx  ← regenerated
01-franchise-module/FLOWCHARTS.html
01-franchise-module/FLOW_INTERACTIVE.html
01-franchise-module/MODULES.md
02-product-module/PRODUCT_INTERACTIVE.html
02-product-module/PRODUCT_UI_CLIENT.html
03-billing-module/BILLING_INTERACTIVE.html
05-point-incentive-module/POINT_INCENTIVE_INTERACTIVE.html
QUESTIONS.md                                          ← open questions
CHANGES_STRUCTURE.md                                  ← ye file
```

**Untouched:** `04-user-module/USER_MODULE_INTERACTIVE.html` — chain rule pehle se sahi tha.

**Note:** `00-documents/` ke andar `Apolium_System_Overview_CLIENT.docx/.pdf` aur `Apolium_Technical_Architecture_DEV.docx/.pdf` **abhi purane hain** — inka source script repo me nahi hai, isliye regenerate nahi kar paye. Live doc `index.html` hai, wo updated hai.
