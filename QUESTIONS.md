# Client Questions — Status

> Last update: 2026-09-01 (client meeting #2 ke baad)
> **Saare pichhle sawaal ab confirm ho chuke hain.** Neeche jawab hain.
> Kya-kya badla: [CHANGES_STRUCTURE.md](CHANGES_STRUCTURE.md)

---

## ✅ Ab confirm ho chuka

### Q1. Nayi structure me commission split kaise hoga?

**Jawab:** L2 aur L3 ke beech **koi rishta nahi hai**. Baaki flow bilkul pehle jaisa — har level ko **apna set commission** milta hai.

| User ne khareeda | Kis-kis ko commission |
|---|---|
| Level 2 se | **L1 + L2** — dono ko apna-apna |
| Level 3 se | **L1 + L3** — dono ko apna-apna |
| Level 1 se (direct) | L1 ko apna, baaki 50% Warehouse ko (50/50) |

Ek raaste ka level doosre raaste me aata hi nahi, isliye aapas me kuch bantta bhi nahi.
Commission ka number waise bhi **product master se** aata hai — Admin har product pe har level ka rate set karta hai.

### Q2. "Missing level ka roll-up" rule?

**Jawab: ✅ Khatam ho gaya.** L2 aur L3 alag branch hain — ek ka na hona doosre ko affect hi nahi karta. Beech me koi "gap" ban hi nahi sakta.

### Q3. L1 (postpaid) ke billing ka paisa kahan jaata hai?

**Jawab:** **L1 kabhi real money pehle deta hi nahi** — wo postpaid hai aur uske paas limit hoti hai. Uska wallet **minus aur plus hota rehta hai**, bilkul jaise pehle chal raha tha. Koi naya behaviour nahi.

### Q4. L1 ki credit limit — fix hai ya per-franchise?

**Jawab:** **Per franchise.** Admin har L1 ke liye alag set karta hai. (₹15 lakh sirf ek example tha.)

### Q5. Achievers — Direct aur Direct Team

**Jawab:** Ye **do alag cheezein** hain — sheet me do alag column hain. (Pehle humne galti se dono ko ek hi column me mila diya tha.)

| Lv | Rank | Left team PV<br>(Org-1) | Right team PV<br>(Org-2) | **Direct**<br>(ek baar) | **Direct Team**<br>(har mahine) | Reward | % Turnover |
|---|---|---|---|---|---|---|---|
| 1 | Silver Director | 1,00,000 | 1,00,000 | 5,000 | **25,000** | 3N4D Goa stay | 15% |
| 2 | Gold Director | 5,00,000 | 5,00,000 | 25,000 | **50,000** | Domestic Tour | 10% |
| 3 | Platinum Director | 25,00,000 | 25,00,000 | 50,000 | **1,00,000** | Intl. Tour + 2% car fund | 8% |
| 4 | Diamond Director | 50,00,000 | 50,00,000 | 50,000 | **2,00,000** | International Tour | 7% |
| 5 | Crown Director | 2,00,00,000 | 2,00,00,000 | 50,000 | **5,00,000** | Intl. Family Tour (1+1) + 3% house fund | 7% |

- **Org-1 = Left team PV** · **Org-2 = Right team PV** (binary tree ki dono legs)
- **Direct** ("Sales by Direct Joiners") = **EK BAAR**. Jinko khud sponsor kiya unki sale. Target pura ho gaya → **baat khatam**, dobara nahi dekha jaata
- **Direct Team** ("Direct Team Sales") = **HAR MAHINE**. Jaise Silver pe ho → us mahine **25,000 PV** ka direct-team business karna hi padega, **tabhi agle mahine** us rank ka benefit milega

### Q6. Startup Incentive ki time-limit window kitni?

**Jawab:** **Admin ke haath me.** Jitne din ya mahine chahe rakh sakta hai, aur baad me badal bhi sakta hai. System me koi fix number hardcode nahi hoga. (180 din sirf example.)

### Q7. Bill cancel / stock return pe kya hoga?

**Jawab:** **Poora flow reverse ho jaayega.** Jo stock gaya tha wo wapas, jo commission bana tha wo wapas — sab kuch ulta chal jaayega. Reverse ka bhi record rehta hai.

### Q8. Payout kaise jaayega?

**Jawab:** Payout **mahine me ek baar**. **TDS 2%** katega.

---

## ✅ Carry Forward & Washout — poora rule (sir ne example ke saath samjhaya)

**Strong / weak rule CHALU hai** — hataya nahi gaya.

1. Dono side ka **total = purana carry + aaj ka business**
2. Matching = min(totalL, totalR), 1,000 ke multiple me neeche round, daily cap tak
3. **Strong side** = jiska **total** bada · **Weak side** = jiska **total** chhota — *(sir ne confirm kiya: strong/weak **total se hi** decide hota hai, sirf aaj ke business se ya sirf carry se nahi)*
4. **Matching pehle AAJ ka business kharch karti hai**, uske baad purana carry — *(ye order sabse zaroori hai)*
5. **Purana carry jo bacha → hamesha aage.** Old carry kabhi wash nahi hota
6. **Aaj ke business ka jo bacha → wahi wash hota hai**
7. **Weak side pe washout nahi hua → strong side pe bhi nahi hoga**
8. Matching bani hi nahi → kuch wash nahi, dono side pura carry

### Sir ke chaaro example — code se verify ho chuke

| # | Left (carry + aaj) | Right (carry + aaj) | Match | Result |
|---|---|---|---|---|
| 1 | 0 + 500 = 500 | 0 + 300 = 300 | 0 | Matching bani hi nahi → carry **500 / 300**, koi wash nahi |
| 2 | 300 + 800 = 1,100 | 500 + 1,000 = 1,500 | 1,000 | Dono ka aaj ka business match me laga → **koi wash nahi**. Carry **100 / 500** |
| 3 | 2,000 + 0 = 2,000 | 900 + 100 = 1,000 | 1,000 | Carry **1,000 / 0**. Koi wash nahi |
| 4 | 2,000 + 800 = 2,800 | 0 + 1,200 = 1,200 | 1,000 | Strong = Left. L ka aaj ka 800 pura laga → carry **1,800**. R ka aaj ka **200 wash** |

### Purani sheet se bhi match

| Din | Old L | Old R | Aaj R | Match | Carry L | Carry R | Wash R |
|---|---|---|---|---|---|---|---|
| Day 1 | 1,59,400 | 2,047 | 0 | 2,000 | **1,57,400** | **47** | 0 |
| Day 2 | 1,57,400 | 47 | 2,394 | 2,000 | 1,55,400 | 47 | **394** |

> **Note:** Day-2 ka washout ab **394** aata hai — jo aapki sheet me likha tha.
> Pehle hum galti se 441 keh rahe the, kyunki tab hum maan rahe the ki matching pehle purana carry kharch karti hai.
> Ab order theek hai: **pehle aaj ka business, phir carry.**

---

## ✅ Pichhli meeting ke confirmed points (waise hi hain)

1. **L1 Special / koi bhi special level — hata diya.** Sirf normal L1, L2, L3
2. **L2 aur L3 dono seedha L1 ke neeche** (siblings). Stock lene ka flow same
3. **Pincode rollup:** L2 ko ab L3 ke pincode nahi milte, dono ka rollup seedha L1 me
4. **L1 = POSTPAID** (credit limit) · **L2/L3 = PREPAID** (wallet recharge). Do bilkul alag flow
5. **Paisa upar wallet-to-wallet transfer se** jaata hai, cash nahi
6. **Price aur commission fix nahi** — product master se aate hain
7. **Billing ke waqt cost price customize** ho sakti hai
8. **Chain rule:** left ka left, right ka right. Sponsor ki neeche wali chain me hi
9. **Point 4 tarah ke:** Self · Direct · Direct Team · Team
10. **"Admin special limit"** (buying limit override) — alag cheez, untouched

---

## 🔶 Ab bhi baaki (chhoti cheezein)

1. **CV ka use** — PV ke rules clear hain, GV Achiever pool ke turnover me use hota hai. **CV exactly kahan lagega** — abhi define karna hai.
2. **Cap-out vs washout** — dono alag mechanism hain, iska detail flow aana baaki hai.
