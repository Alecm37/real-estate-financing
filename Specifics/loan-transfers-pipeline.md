# Service 4: Loan Transfers (Assumable Mortgages), explained as a pipeline

*Market backdrop: mid-October 2026. All loan-level and property-level numbers below are one worked example run end to end through simulations; they are illustrative, not quotes of any real borrower, servicer, or sale.*

---

## 0. What we are valuing, and why it needs five models

### The object being valued

When most homes sell, the seller's mortgage is paid off at closing and the buyer takes out a new loan at today's rate. A few kinds of loans can instead be **assumed**: the buyer takes over the seller's existing loan, keeping its remaining **balance**, its **note rate** (the interest rate written in the promissory note), and its remaining **term** (months left until the loan is fully paid). The seller walks away with the difference between the sale price and that balance.

Which loans are assumable is set by law and by the loan contract:

- **FHA loans** (insured by the Federal Housing Administration, typically low down payments and lower credit scores) are assumable by a buyer who passes the servicer's creditworthiness review.
- **VA loans** (guaranteed by the Department of Veterans Affairs, for veterans and service members) are assumable with the servicer's approval. A non-veteran can assume one, but the seller's VA **entitlement** (their remaining guarantee capacity for a future VA loan) stays tied up in the assumed loan unless the buyer is a veteran who substitutes their own.
- **USDA loans** (rural housing loans) are also assumable, with conditions.
- **Conventional loans** (Fannie Mae, Freddie Mac, and bank portfolio loans) almost never are: they carry a **due-on-sale clause**, enforceable under the Garn–St Germain Act of 1982, which makes the whole balance due when the home is sold.

Nearly every FHA, VA and USDA loan is pooled into securities guaranteed by **Ginnie Mae** (the Government National Mortgage Association). That fact is what makes this service possible, because Ginnie Mae publishes loan-level data on those pools.

The **servicer** is the company that collects payments and runs the loan day to day; for a Ginnie Mae loan it is the pool's **issuer**. The servicer, not the original lender, processes an assumption.

The key complication is the **equity gap**: the sale price minus the assumed balance. A buyer can assume a \$283,000 loan, but if the house sells for \$427,000 the seller still has to be paid \$144,000, either in cash or with a **second lien** (a second mortgage recorded behind the assumed loan), which is priced at today's high rates.

The quantity this service produces, for each property $i$ carrying an assumable loan, is a set of answers to four questions:

1. **Where are these loans?** Which homes carry an assumable low-rate loan, with what probability the match is right.
2. **What is the loan worth?** The realistic value to a buyer of taking over the loan instead of borrowing at today's rate, as a distribution over interest-rate paths.
3. **Does the market pay for it?** How much more an assumable home sells for than a comparable one, and what share of the loan's value that premium represents.
4. **Can the deal close?** The buyer's effective rate and cash needed, and the probability and timing of closing.

### Why it is worth doing now

- The 30-year fixed rate is near 7.3% (Freddie Mac PMMS) after the late-September oil-shock spike, while about half of outstanding mortgages carry rates below 4%. Every one of those low-rate FHA and VA loans is a below-market financing contract attached to a house.
- That gap is the **lock-in effect**: owners with cheap loans don't sell, because selling means giving up the loan. An assumable loan is the one case where the cheap loan can be sold *with* the house.
- Yet assumptions remain rare. Sellers and agents often don't know their loan is assumable, buyers can't see it in listing data, servicers process assumptions slowly, and the equity gap is large. That gap between the value that exists and the value that changes hands is the business opportunity.
- FHA loans carry the concentrated delinquency stress in this market (delinquencies about 4.37% and rising). A creditworthy buyer assuming a stretched seller's FHA loan is a better outcome for the servicer and Ginnie Mae than a distressed sale.

### Why five models instead of one

Each question needs a different tool, and each answer feeds the next:

| Step | Model | Produces |
|---|---|---|
| 4.1 | Probabilistic record linkage (Fellegi–Sunter) + text classification | Which parcels carry which assumable loans, with match probabilities |
| 4.2 | Amortization with a curtailment band | The balance the buyer would assume on the closing date |
| 4.3 | Survival model + Hull-White rate simulation | Realistic value of the loan to a buyer |
| 4.4 | Propensity score matching | The market's price premium and the capture ratio |
| 4.5 | Effective-rate math + competing-risks survival (Kaplan–Meier, Cox) | Buyer's effective rate, cash needed, closing probability and timing |

The naive alternative is one formula: the present value of the payment difference between the old rate and today's rate. As we will see, that number (about \$90,000 for our example loan) overstates the realistic value (about \$51,000) by roughly three-quarters, because buyers move or refinance long before the loan matures, and rates may fall.

### The actual flow of numbers

```
Ginnie Mae loan-level data ─┐
                            ├─► 4.1 record linkage ──► loan L ↔ parcel i, match prob. P
County recorder documents ──┘        │ (terms, servicer, loan type)
                                     ▼
                          4.2 amortization ──► balance at closing B^close ──────────────┐
                                     │                                                  │
Rate simulator (Hull-White) ─┐       ▼                                                  │
Turnover hazard (5.2) ───────┴► 4.3 valuation ──► loan value V^loan ─┐                  │
                                                                     ├─► capture ratio ─┤
Sales + recorder data ────────► 4.4 propensity matching ─► premium Π ┘                  │
AVM (3.1) ──► market value of home ─────────────────────────────────────────────────────┤
                                                                                        ▼
                     4.5 deal feasibility: equity gap, second lien, effective rate, time to close
                                                                                        │
                                                                                        ▼
                                                                                     report
```

Three links deserve emphasis. First, 4.1 hands off not only the loan's terms but also its **servicer** (the Ginnie Mae issuer), which is the single strongest predictor of how fast an assumption closes in 4.5. Second, 4.3 and 4.4 measure the same thing from two sides: 4.3 says what the loan *should* be worth to a buyer, 4.4 says what buyers *actually* pay; their ratio is the **capture ratio**, the central diagnostic of the service. Third, the equity gap in 4.5 feeds back into the value: money the buyer borrows at 8.5% to cover the gap erodes part of the benefit of the 3% loan.

### Timeline used in the worked example

- **Today:** mid-October 2026. The seller is about to list.
- **Listing:** November 2026. Contract signed around December 1, 2026.
- **Assumption closing:** targeted for March 2027, after the servicer's review. The buyer's first payment on the assumed loan is April 2027.
- All valuations are as of the closing date; month $t = 1$ is the first month after closing.

### The running example: House H with Loan L

A three-bedroom house in Albemarle County, Virginia (outside Charlottesville).

- The current owners bought in **December 2020** for \$342,000 with an **FHA loan** of **\$330,000** at **3.00%**, 30-year fixed, first payment February 1, 2021. Monthly principal and interest: **\$1,391.29**.
- Because the original loan-to-value was above 90%, the loan carries FHA's **annual mortgage insurance premium (MIP)** of **0.85%** of the balance for the life of the loan. (MIP is FHA's insurance charge, paid monthly on top of principal and interest.)
- The AVM (3.1) puts the January 2027 market value at a median of **\$415,000**.
- The owners are relocating and want to know: is my loan worth marketing, what should I ask, and will a buyer actually be able to close?

The punchline we will derive: the loan is realistically worth about **\$51,000** to a buyer (not the \$90,000 a naive calculation suggests); the market typically pays about **27%** of that value as a price premium, about **\$14,000**; and a buyer with \$44,000 in cash and a \$100,000 second lien gets an all-in effective rate of about **5.35%** instead of **7.55%**, saving about **\$276 a month**. But the deal has only about a **45%** chance of closing within 120 days, and a one-in-three chance of falling through altogether, mainly because of the equity gap and the servicer.

---

## Step 4.1: Identification (probabilistic record linkage + text classification)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Ginnie Mae loan-level records: original amount, note rate, first payment date, term, loan purpose, agency (FHA/VA/USDA), loan-to-value, number of borrowers, issuer (servicer), current unpaid balance, state. No street address. | record $g$ | Ginnie Mae monthly loan-level disclosure files |
| County recorder documents: deeds of trust (mortgages) with recording date, loan amount, lender name, borrower names, maturity date, property address and parcel ID; deeds with sale price | document $d$ | County recorder (in Virginia, the Circuit Court Clerk's land records) |
| Text of each recorded deed of trust (scanned and OCR'd) | $\mathbf{z}_d$ | Same, via image or text purchase |
| Releases and satisfactions (documents recorded when a loan is paid off) | | Same |

**Terms.** A **deed of trust** is the document, recorded in the county land records, that pledges the house as security for the loan (Virginia uses deeds of trust rather than mortgages; the role is the same). **OCR** (optical character recognition) turns scanned page images into text. A **parcel ID** is the county's identifier for a lot. **Record linkage** means deciding which record in one database refers to the same real-world object as a record in another, when no shared ID exists.

### Why linkage is needed

This is the cross-cutting constraint of the whole project made concrete: Ginnie Mae tells us a loan exists, with its exact terms and current balance, but not *where*. The county knows *where* a loan was recorded and on which parcel, but usually not its rate, its current balance, or whether it is still outstanding. Neither alone answers "which homes carry a sub-4% assumable loan." We have to join them without a common key, so every link is a probability.

### Process, part 1: labeling loan type from document text

The comparison in part 2 needs to know whether each recorded deed of trust is FHA, VA, or conventional. Deeds don't have a clean field for it, so we read it from the text.

**Rules first.** Most FHA and VA deeds of trust carry tell-tale strings: "FHA Case No." in the header, references to the National Housing Act, a "VA Guaranteed Loan and Assumption Policy Rider," a "VA Loan Number." Conventional deeds usually carry the Fannie Mae/Freddie Mac uniform instrument footer. A regular-expression rule set labels the document whenever exactly one family of strings appears.

**Classifier for the rest.** When OCR is poor or the strings are missing, a **multinomial logistic regression** on word and character n-gram features of the text estimates

$$q_{d,k} = \Pr(\text{type}_d = k \mid \mathbf{z}_d) = \frac{\exp(\mathbf{z}_d'\boldsymbol{\omega}_k)}{\sum_{k'} \exp(\mathbf{z}_d'\boldsymbol{\omega}_{k'})}, \qquad k \in \{\text{FHA}, \text{VA}, \text{conv}\},$$

where $\mathbf{z}_d$ is the document's feature vector (counts of words and short character sequences, scaled by how rare they are across documents, called TF-IDF) and $\boldsymbol{\omega}_k$ the coefficient vector for type $k$. The training labels come from the rule-labeled documents plus documents that matched a Ginnie Mae record with certainty. We accept the label when the top $q_{d,k} \ge 0.9$ and otherwise leave the field "unknown" for the comparison step.

### Process, part 2: Fellegi–Sunter matching

**Blocking.** Comparing every Ginnie record with every recorder document is wasteful, so we only compare pairs within a **block**: the same state and the same origination month. (The Ginnie record's first payment date is normally the first of the second month after closing, so a first payment in February 2021 implies a December 2020 closing.)

**The comparison vector.** For each candidate pair $(g, d)$ in a block, compare $F$ fields and record agreement:

$$\boldsymbol{\gamma}_{gd} = \big(\gamma^{1}_{gd}, \dots, \gamma^{F}_{gd}\big), \qquad \gamma^{f}_{gd} = \begin{cases} 1 & \text{if field } f \text{ agrees} \\ 0 & \text{if it disagrees.} \end{cases}$$

The fields here are: original amount (exact to the dollar), closing month, loan type (from part 1), lender name (fuzzy string similarity of at least 0.9), loan purpose (purchase if a sale deed was recorded the same day), maturity date, and loan-to-value (Ginnie's LTV vs. loan amount divided by the deed's sale price, within half a point).

**The two probabilities per field.** Let $M$ be the set of true matches and $U$ the set of non-matches. For each field,

$$m_f = \Pr\big(\gamma^f = 1 \mid (g,d) \in M\big), \qquad u_f = \Pr\big(\gamma^f = 1 \mid (g,d) \in U\big).$$

$m_f$ is a field's **reliability**: even for a true match, the amount can be keyed wrong or the lender recorded under a broker's name, so $m_f < 1$. $u_f$ is its **coincidence rate**: how often two unrelated loans agree by chance. A field with $u_f$ near zero (exact loan amount) is powerful; a field most loans share (30-year maturity) is weak.

**Field weights.** Assuming fields agree independently given match status, the log likelihood ratio of a comparison vector is a sum of field weights:

$$W_{gd} = \sum_{f=1}^{F} w_f, \qquad w_f = \begin{cases} \log\dfrac{m_f}{u_f} & \text{if } \gamma^f_{gd} = 1, \\[2ex] \log\dfrac{1 - m_f}{1 - u_f} & \text{if } \gamma^f_{gd} = 0. \end{cases}$$

Agreement adds a positive weight, disagreement a negative one. (We use natural logs so the posterior below is a logistic function.)

**From weight to probability.** Let $\pi$ be the share of pairs in the block that are true matches. By Bayes' rule,

$$\log \frac{P_{gd}}{1 - P_{gd}} = \log\frac{\pi}{1-\pi} + W_{gd}, \qquad P_{gd} = \Pr\big((g,d) \in M \mid \boldsymbol{\gamma}_{gd}\big).$$

**Estimating $m_f$, $u_f$, $\pi$ without labels: the EM algorithm.** We don't know which pairs are matches, so we treat match status as a hidden variable. Start from rough guesses, then alternate:

- **E-step:** compute each pair's current match probability $P_{gd}$ from the formula above.
- **M-step:** re-estimate the parameters as probability-weighted frequencies:

$$\hat m_f = \frac{\sum_{gd} P_{gd}\,\gamma^f_{gd}}{\sum_{gd} P_{gd}}, \qquad \hat u_f = \frac{\sum_{gd} (1 - P_{gd})\,\gamma^f_{gd}}{\sum_{gd} (1 - P_{gd})}, \qquad \hat\pi = \frac{1}{|\text{pairs}|}\sum_{gd} P_{gd}.$$

Repeat until the parameters stop changing. Each iteration provably does not decrease the likelihood of the observed agreement patterns.

**Decision rule and one-to-one assignment.** Accept links with $P_{gd} \ge t_U$ (upper threshold, here 0.90), reject below $t_L$ (lower, here 0.10), and send the band in between to **clerical review** (a person looks at the documents). Finally, a loan can sit on only one parcel, so among accepted links we solve a one-to-one assignment that maximizes total weight (the Hungarian algorithm, or greedy by weight in practice).

**Is the loan still outstanding and still assumable?** A linked loan is kept only if (a) it appears in the current month's Ginnie file (still in a pool, balance positive), (b) no release or satisfaction for that deed of trust has been recorded, and (c) no later deed shows a sale with a new deed of trust (which would mean it was paid off).

### Worked example

**The block.** Virginia, December 2020 closings: 3,500 Ginnie records against the 240 deeds of trust recorded that month in the counties pulled (Albemarle and Charlottesville city), so $840{,}000$ candidate pairs. EM estimates about 150 true matches, so $\hat\pi = 150 / 840{,}000 = 0.000179$ and the prior log odds are $\log(0.000179/0.999821) = -8.63$. Without evidence, any given pair is almost surely not a match.

**EM estimates and weights:**

| Field | $m_f$ | $u_f$ | Agree weight $\log(m/u)$ | Disagree weight $\log\frac{1-m}{1-u}$ |
|---|---|---|---|---|
| Original amount (exact) | 0.96 | 0.004 | +5.48 | −3.22 |
| Closing month | 0.90 | 0.30 | +1.10 | −1.95 |
| Loan type | 0.97 | 0.20 | +1.58 | −3.28 |
| Lender name (fuzzy) | 0.82 | 0.07 | +2.46 | −1.64 |
| Loan purpose | 0.95 | 0.60 | +0.46 | −2.08 |
| Maturity date | 0.98 | 0.80 | +0.20 | −2.30 |
| Loan-to-value | 0.93 | 0.15 | +1.82 | −2.50 |

**Loan L's candidate pairs.** The Ginnie record: FHA, \$330,000, 3.000%, first payment 02/2021, 360 months, purchase, LTV 96.5%, issuer "Blue Ridge Home Lending" (a placeholder name). Three recorder documents in the block share some of its features:

| Candidate | What agrees | $W$ | Posterior log odds | $P_{gd}$ |
|---|---|---|---|---|
| $d_1$: deed of trust on House H, \$330,000, recorded Dec 18, 2020, lender "Blue Ridge Home Lending LLC," FHA Case No. in header, maturity 01/01/2051, same-day sale deed at \$342,000 | all seven | 13.11 | $-8.63 + 13.11 = 4.48$ | **0.989** |
| $d_2$: also \$330,000, but recorded November, different lender, LTV 80% | amount, type, purpose, maturity | 1.64 | −6.99 | 0.001 |
| $d_3$: \$329,500, otherwise agrees | all but amount | 4.41 | −4.22 | 0.014 |

Only $d_1$ passes the upper threshold. The power comes mainly from the exact amount: agreement on \$330,000 alone multiplies the odds by $e^{5.48} \approx 240$, because only 0.4% of unrelated pairs share an exact amount. Notice how much a single disagreement costs: had the deed listed the loan under a mortgage broker's name instead of the lender's, the lender field would flip to −1.64 and the total would fall to $13.11 - 2.46 - 1.64 = 9.00$, giving $P = 0.59$, squarely in the clerical-review band.

**Type label.** $d_1$'s header carried "FHA Case No." so the rule fired. For comparison, a deed in the same block whose first page OCR'd badly got a classifier probability $q_{\text{FHA}} = 0.94$, which also passes.

**Still active?** Loan L appears in the September 2026 Ginnie file with a reported unpaid balance of \$286,882, and no release has been recorded. It is a live, assumable, 3.00% FHA loan on House H.

### Output

For each parcel: a list of linked assumable loans with match probability $P_{gd}$, the Ginnie record's terms (amount, rate, first payment date, term, agency, MIP-relevant LTV), the reported current balance and as-of month, the servicer, and the active flag. At the market level: counts of active sub-4% FHA/VA loans by locality.

### Hand-off

Loan terms and the reported balance go to 4.2. The parcel ID goes to the AVM (3.1) for the home's market value. The servicer name and loan type go to 4.5 as covariates in the time-to-close model. The full set of linked loans defines the "treated" population that 4.4 can study. The same linkage machinery is reused in Services 2 and 3 for parcel-to-loan joins.

### Validation

Hold out a sample of links and verify them by hand (the documents make it unambiguous once a person reads them); measure precision (share of accepted links that are right) and recall (share of true matches found) by threshold. Check that the EM-estimated $\hat\pi$ agrees with the clerical sample. Re-run EM per state, since recording practices differ.

---

## Step 4.2: Balance reconstruction (amortization with a curtailment band)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Original amount | $B_0$ | Ginnie record (4.1), confirmed by the deed of trust |
| Note rate (annual) | $c$ | Ginnie record |
| Original term in months | $n$ | Ginnie record |
| First payment date | | Ginnie record |
| Reported current balance and its as-of month, when available | $\tilde B_{k_0}$ | Ginnie monthly file (4.1) |
| Distribution of cumulative **curtailments** by loan age | | Agency loan-level performance data |
| Target closing date | | Contract (4.5) |

**Terms.** **Amortization** is the schedule by which fixed monthly payments pay interest first and then reduce principal. A **curtailment** is an extra principal payment beyond the scheduled amount. On a fixed-rate loan, a curtailment does not lower the monthly payment; it makes the loan end sooner.

### Why this matters more than it looks

Every dollar of balance error is a dollar of cash the buyer must find or borrow at second-lien rates, because the equity gap is price minus balance. A \$5,000 error in a balance is a \$5,000 error in the buyer's cash requirement and changes the deal.

### Process

With monthly rate $\rho = c/12$, the fixed payment is

$$\text{PMT} = B_0 \,\frac{\rho}{1 - (1+\rho)^{-n}},$$

and the scheduled balance after $k$ payments is

$$B_k = B_0 (1+\rho)^k - \text{PMT}\,\frac{(1+\rho)^k - 1}{\rho}.$$

Read the formula as "what the original balance would have grown to at the note rate, minus what the payments made would have grown to." Each month, interest is $\rho B_{k-1}$ and the principal paid is $\text{PMT} - \rho B_{k-1}$, so principal reduction accelerates as the balance falls.

**Two modes.**

1. **Anchored** (when the matched Ginnie record reports a current balance $\tilde B_{k_0}$ after $k_0$ payments). Any gap between schedule and report is curtailment already made, $X = B_{k_0} - \tilde B_{k_0}$. Because the payment does not change, that gap grows at the note rate, so at closing after $k_1$ payments
$$B^{\text{close}} = B_{k_1} - X(1+\rho)^{k_1 - k_0}.$$
2. **Unanchored** (match below threshold, or a loan not in a Ginnie pool). Use the scheduled balance as the upper edge and subtract a high percentile of the curtailment distribution for loans of that age to get a lower edge. Report the band, not a point.

### Worked example

$B_0 = \$330{,}000$, $c = 3.00\%$, $\rho = 0.0025$, $n = 360$:

$$\text{PMT} = 330{,}000 \times \frac{0.0025}{1 - 1.0025^{-360}} = \$1{,}391.29.$$

**Anchor.** The September 2026 Ginnie file reports \$286,882 after $k_0 = 68$ payments (February 2021 through September 2026). The schedule says

$$B_{68} = 330{,}000 \times 1.0025^{68} - 1{,}391.29 \times \frac{1.0025^{68} - 1}{0.0025} = \$288{,}082.19,$$

so the owners have made about $X = \$1{,}200$ of extra principal payments.

**Projection to closing.** The March 2027 payment is number $k_1 = 74$. $B_{74} = \$284{,}030.42$, and

$$B^{\text{close}} = 284{,}030.42 - 1{,}200.19 \times 1.0025^{6} = 284{,}030.42 - 1{,}218.31 = \$282{,}812.11.$$

The remaining schedule runs **285 months** (one fewer than 286 because of the curtailment), with a small final payment.

**How fast equity builds.** The first post-closing payment splits into interest $0.0025 \times 282{,}812 = \$707.03$ and principal $1{,}391.29 - 707.03 = \$684.26$. Nearly half of each payment reduces principal: a low-rate loan amortizes fast, which matters for both the value in 4.3 (the balance the discount applies to shrinks) and the buyer's equity.

**Unanchored band, for comparison.** Without the Ginnie report, the scheduled \$284,030 is the top of the band; the 90th percentile of cumulative curtailment at loan age six years in agency data is about 1.9% of the original amount, or \$6,270, putting the bottom near \$277,800. A \$6,000 band would be acceptable for marketing but not for a contract, which is why the anchor is worth having.

### Output

$B^{\text{close}}$ (or a band), the full forward balance path $B_1, B_2, \dots, B_T$ from closing, the payment PMT, and the remaining term $T$.

### Hand-off

The balance path goes to 4.3 (savings are interest-rate differences applied to this balance). $B^{\text{close}}$ goes to 4.5 to size the equity gap.

### Validation

On loans with Ginnie-reported balances, predict the balance from terms alone and compare: the scheduled balance should sit on or above the reported balance (curtailments only lower it), and the curtailment band's coverage should match its nominal level.

---

## Step 4.3: Valuation (survival model + Hull-White rate simulation)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Balance path of the assumed loan | $B_{t}$, $t = 0, \dots, T$ | 4.2 |
| Note rate and MIP rate of the assumed loan | $c$, $\mu^{\text{A}}$ | Ginnie record (4.1); FHA MIP schedule |
| Today's rate on a fresh loan of the same type, and its MIP rate | $m_0$, $\mu^{\text{F}}$ | PMMS / FHA rate surveys; FHA MIP schedule |
| Today's Treasury yield curve | | Fed GSW zero curve |
| Rate volatility | $\sigma$ | Swaption implied volatility or history of monthly changes |
| Household exit hazard: sale, default, other payoff | $h_t$ | Turnover hazard model, step 5.2 (shared module, trained on Fannie Mae and Freddie Mac loan-level data) |
| Refinancing threshold and cost | $\kappa$, $k^{\text{R}}$ | Literature on optimal refinancing; closing-cost surveys |

### What "value of the loan" means

The cleanest definition: the value of a below-market loan to the borrower is the amount they save by holding it instead of borrowing the same balance at the market rate. Take the assumed loan's balance path $B_t$ and imagine a second loan with *exactly the same balance path* but at the fresh-loan rate. Each month both loans repay the same principal, so the only difference in payments is interest (plus mortgage insurance):

$$s_t = \frac{\big(c^{\text{F}}_t + \mu^{\text{F}}\big) - \big(c^{\text{A}}_t + \mu^{\text{A}}\big)}{12}\; B_{t-1},$$

where $c^{\text{F}}_t$ is the rate the fresh-loan borrower would be paying in month $t$ and $c^{\text{A}}_t$ the rate the assumer is paying. This "same balance path" comparison is exact for the naive case: discounting $s_t$ at a constant market rate $m$ over the full term gives precisely $B_0$ minus the present value of the assumed loan's payments at rate $m$, which is the familiar payment-difference formula. (Check: \$300,000 at 3% vs. 7% over 300 months gives \$98,700, the naive figure in the project context.)

The naive value assumes three things that are all false: the buyer keeps the loan to maturity, rates stay where they are, and nobody refinances. Step 4.3 relaxes all three.

### Process, part 1: rate paths (Hull-White)

**The model.** The **short rate** $r_t$ (the instantaneous risk-free rate) follows

$$dr_t = \big(\theta(t) - a\, r_t\big)\,dt + \sigma\, dW_t,$$

where $a$ is the speed of **mean reversion** (how fast rates are pulled back toward their expected path), $\sigma$ the volatility, $dW_t$ a Brownian motion increment (normal noise with variance $dt$), and $\theta(t)$ a time-dependent drift **calibrated** so that the model reproduces today's Treasury zero curve exactly. That calibration is what distinguishes Hull-White from the simpler Vasicek model: it starts from the market's own curve rather than a guessed long-run level.

**The convenient decomposition.** Write $r_t = \varphi(t) + x_t$, where $\varphi(t)$ is the deterministic part fixed by the curve and $x_t$ is a mean-reverting factor with $x_0 = 0$:

$$dx_t = -a\,x_t\,dt + \sigma\, dW_t.$$

In monthly steps ($\Delta = 1/12$) this is exactly an AR(1):

$$x_{t+1} = e^{-a\Delta} x_t + \sigma\sqrt{\frac{1 - e^{-2a\Delta}}{2a}}\;\varepsilon_{t+1}, \qquad \varepsilon_{t+1} \sim \mathcal{N}(0,1).$$

This is the same structure as the Labor Cost simulator (1.5), now with its parameters tied to an arbitrage-free model.

**From the short rate to the mortgage rate.** Mortgage rates track long yields (roughly the 10-year Treasury), not the short rate. In Hull-White every zero-coupon yield is linear in $x_t$; the $\tau$-year yield moves by $B(\tau)/\tau$ per unit of $x_t$, where $B(\tau) = (1 - e^{-a\tau})/a$. So the fresh-loan rate on path $n$ is

$$m_{n,t} = f^{m}_t + b_{10}\, x_{n,t}, \qquad b_{10} = \frac{1 - e^{-10a}}{10a},$$

where $f^{m}_t$ is the fresh-loan rate implied by today's forward curve plus the mortgage spread. With mean reversion, long rates move less than short rates ($b_{10} < 1$).

**Refinancing on each path.** Both the assumer and the fresh-loan borrower refinance when it pays. We use a threshold rule as the practical stand-in for the optimal policy: refinance in month $t$ if the market rate is at least $\kappa$ below the current rate, paying a cost of $k^{\text{R}}$ times the balance. (The optimal threshold, as derived by Agarwal, Driscoll and Laibson, depends on rate volatility, refinancing costs, and how long the borrower expects to stay; for typical values it falls near one percentage point.)

- The fresh-loan borrower starts at $c^{\text{F}}_0 = m_0$ and resets to $m_{n,t}$ whenever $m_{n,t} \le c^{\text{F}}_{t-1} - \kappa$. Each such refinance costs them $k^{\text{R}} B_{t-1}$, a cost the assumer avoids, so it is **added** to $s_t$ in that month.
- The assumer refinances only if $m_{n,t} + \mu^{\text{F}} \le c + \mu^{\text{A}} - \kappa$, which at a 3% note rate essentially never happens.

Refinancing is the reason rate *volatility* lowers the loan's value: when rates fall, the fresh-loan borrower catches up and the savings shrink; when rates rise, the fresh-loan borrower is stuck at the rate they locked, so savings do not grow. The fresh borrower holds a free option the assumer doesn't need.

### Process, part 2: how long the buyer keeps the loan (survival)

The savings last only while the buyer holds the loan. The assumed loan ends when the household sells (moving), defaults, or pays off for other reasons (a cash-out refinance, divorce, inheritance). Let $h_t$ be the **hazard**: the probability the loan ends in month $t$ given it survived to $t-1$. The **survival function** is

$$S(t) = \prod_{u=1}^{t} \big(1 - h_u\big).$$

The hazard comes from the turnover model in 5.2, which is a competing-risks model fit on a loan-month panel. For this loan we use its pieces:

$$h_t = h^{\text{sale}}_t + h^{\text{def}} + h^{\text{oth}}, \qquad h^{\text{sale}}_t = \frac{\text{ten}(t)}{1 + \exp\!\big(-(\eta_0 - \theta_g\, G_t)\big)}.$$

- $\eta_0$ is the baseline log odds of a sale in a month.
- $G_t = m_{n,t} - c^{\text{A}}_t$ is the **lock-in gap** in percentage points: how far the market rate sits above the buyer's own rate. $\theta_g > 0$ means a bigger gap lowers the chance of moving. This is the lock-in effect, estimated, and it matters here: the assumer is *more* locked in than an ordinary buyer, so they stay longer.
- $\text{ten}(t)$ is a **tenure** multiplier: new owners rarely sell in their first year or two.
- $h^{\text{def}}$ and $h^{\text{oth}}$ are the default and other-payoff hazards.

Because $G_t$ depends on the rate path, $S$ is path-specific: $S_n(t)$.

### Process, part 3: putting it together

On path $n$, discount each month's savings at the path's fresh-loan rate (the borrower's opportunity cost of money: the rate they would otherwise borrow at),

$$\text{DF}_{n,t} = \prod_{u=1}^{t} \Big(1 + \frac{m_{n,u}}{12}\Big)^{-1},$$

and the value of the loan is

$$V^{\text{loan}} = \frac{1}{N}\sum_{n=1}^{N} \; \sum_{t=1}^{T} S_n(t)\; s_{n,t}\; \text{DF}_{n,t}.$$

That is exactly "average over paths of $\sum_t S(t) \times \text{savings}_t \times \text{discount}_t$." The inner sum is the value on one path; the spread of inner sums across paths is the rate risk in the value.

(An investor-style alternative discounts at Treasury rates plus a spread; that gives a higher number. For a buyer deciding what to pay, the fresh-loan rate is the right hurdle.)

**What we leave out, deliberately.** When the buyer later sells, the loan could be assumed *again* by the next buyer, so part of the remaining value could be resold. We exclude that re-assumption option from the base value (conservative) and report it separately when relevant.

### Worked example

**Parameters.**

| Quantity | Value | Comment |
|---|---|---|
| $c$, $\mu^{\text{A}}$ | 3.00%, 0.85% | Loan L; MIP for life |
| $m_0$, $\mu^{\text{F}}$ | 7.00%, 0.55% | Fresh FHA rate (FHA prices a little below the 7.3% conventional PMMS rate); MIP on a new 3.5%-down FHA loan |
| $f^{m}_t$ | 7.00% for all $t$ | Simplification: the fitted curve implies a roughly flat forward mortgage rate |
| $a$, $\sigma$ | 0.05, 0.90%/yr | Slow mean reversion, normal volatility |
| $b_{10}$ | $(1 - e^{-0.5})/0.5 = 0.787$ | 10-year yield moves 0.79 per unit of the factor |
| $\kappa$, $k^{\text{R}}$ | 1.00 point, 1.5% of balance | Refinancing rule |
| $\eta_0$ | −5.033 | Baseline sale probability 0.65%/month, about 7.5%/year at zero gap |
| $\theta_g$ | 0.11 per point | Each point of lock-in gap cuts the odds of moving by 10% |
| $\text{ten}(t)$ | 0.35 (year 1), 0.70 (year 2), 1.0 after | Tenure effect |
| $h^{\text{def}}$, $h^{\text{oth}}$ | 0.08%, 0.10% per month | Default, other payoff |
| $N$ | 20,000 paths | |

**Rate paths.** The monthly factor shock has sd $0.009\sqrt{(1 - e^{-0.1/12})/0.1} = 0.259\%$, so the mortgage rate moves by about $0.787 \times 0.259 = 0.20$ points in a typical month, matching the historical size of monthly PMMS changes. Across paths, the fresh rate's standard deviation is 0.69 points one year out and 1.41 points five years out. There is a 15% chance the rate is below 6% two years out. The fresh-loan borrower refinances at least once in 80% of paths, and within five years in 48%.

**Survival at the central path.** With the gap fixed at $7.00 - 3.00 = 4.0$ points, after the tenure ramp the sale hazard is

$$h^{\text{sale}} = \frac{1}{1 + e^{-(-5.033 - 0.11 \times 4.0)}} = \frac{1}{1 + e^{5.473}} = 0.418\% \text{ per month} \;\; (4.9\% \text{ a year}),$$

and the total exit rate is 6.9% a year. That gives $S(60) = 0.73$, $S(120) = 0.51$, $S(180) = 0.36$: about half of buyers still hold the loan after ten years, and the expected holding time over the 285 remaining months is **142 months (11.9 years)**, not 23.75.

**First month's savings.** On the central path,

$$s_1 = \frac{(7.00\% + 0.55\%) - (3.00\% + 0.85\%)}{12} \times 282{,}812 = \frac{3.70\%}{12} \times 282{,}812 = \$872.00.$$

The MIP difference costs \$70.70 of that month's \$942.71 rate saving: the assumer inherits the old, more expensive mortgage insurance.

**Building up from naive to realistic.** Each row adds one realistic feature:

| Valuation | Value | Share of naive |
|---|---|---|
| Naive: 7.00% vs. 3.00%, held to maturity, no MIP | \$89,969 | 100% |
| + MIP difference (0.55% vs. 0.85%) | \$83,221 | 92% |
| + survival, rates flat | \$58,169 | 65% |
| + stochastic rates and optimal refinancing, no survival | \$68,979 | 77% |
| **Full: survival × rates × refinancing** | **\$51,038** | **57%** |

Survival is the biggest haircut, because most of the naive value sits in years 12 to 24, which most buyers never reach. Rate uncertainty is the second, because the fresh-loan borrower's refinancing option cuts off the savings when rates fall.

**Distribution across rate paths.**

| | P10 | P50 | P90 | Mean |
|---|---|---|---|---|
| $V^{\text{loan}}$ on path | \$41,956 | \$52,757 | \$57,247 | \$51,038 |

The distribution is **skewed left**: the mean is below the median. When rates stay high, the value is capped (the fresh borrower stays at 7%, and the buyer stays put). When rates fall, two things happen at once: the fresh borrower refinances and catches up, *and* the lock-in gap shrinks, so the buyer is more likely to move. In the 35% of paths where the fresh rate dips below 6% within three years, the mean value is \$44,363, versus \$54,634 on the others. Falling rates hurt this asset twice.

### Output

$V^{\text{loan}}$ (mean), its P10/P50/P90 across rate paths, the expected holding time, and the decomposition table above. Per path, the survival curve $S_n(t)$ and savings stream $s_{n,t}$, which 4.5 reuses.

### Hand-off

$V^{\text{loan}}$ goes to 4.4 as the denominator of the capture ratio, and to 4.5 as the gross benefit to a buyer. The path machinery (rates, survival, discounting) is reused in 4.5 to value the second lien. The rate simulator is shared with 5.5.

### Validation

The value itself is never observed, so validate its parts: the survival model's out-of-time calibration on loans of similar vintage and lock-in gap (from 5.2's validation), the rate model's fit to swaption prices or historical volatility, and the refinancing rule against observed refinance timing in agency data. The cross-check that ties it together is 4.4: if the market consistently paid more than $V^{\text{loan}}$, the valuation would be too conservative.

---

## Step 4.4: Market pricing test (propensity score matching)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Home sales: price, date, property characteristics | $Y_i$, $\mathbf{z}_i$ | County recorder deeds and assessor records; MLS or listing aggregates where available |
| Treatment indicator: sold by assumption | $D_i \in \{0,1\}$ | 4.1 linkage: a sale deed with no new deed of trust while the linked FHA/VA loan stays active, or a recorded assumption agreement |
| Loan value for each treated sale | $V^{\text{loan}}_i$ | 4.3 run on each treated loan |

**Terms.** The **treatment** is "sold with an assumption." The **control** group is every other sale. A **propensity score** is the probability a sale is treated given its observed characteristics. The **ATT** (average treatment effect on the treated) is the average price difference for the treated homes compared with what the same homes would have sold for without the assumption.

### Why a raw comparison fails

Homes sold by assumption are not a random sample. They are starter homes bought with FHA or VA loans in 2019 to 2022 (when rates were low): smaller, older, cheaper, farther out. Comparing their average price with all other sales mixes the effect of the loan with the effect of being a cheaper house. We need to compare each assumable sale with non-assumable sales that look the same.

### Process

**Step 1: estimate the propensity score.** A logistic regression of treatment on characteristics:

$$e(\mathbf{z}_i) = \Pr(D_i = 1 \mid \mathbf{z}_i) = \frac{1}{1 + \exp\!\big(-\mathbf{z}_i'\boldsymbol{\alpha}\big)}.$$

The covariates $\mathbf{z}_i$ must be things that affect both whether a home is sold by assumption and its price, measured before the sale: size, age, bedrooms, location, sale quarter, and, importantly, when the seller bought (sellers who bought in 2019–2022 are far more likely to hold a low-rate loan). Never include anything the assumption itself could change, like days on market.

**Why one score is enough.** Rosenbaum and Rubin's result: if treatment is as good as random once we condition on $\mathbf{z}$ (no unmeasured confounders), it is also as good as random once we condition on the single number $e(\mathbf{z})$. Matching on one number is feasible where matching on six characteristics at once is not.

**Step 2: match.** For each treated sale, find the control sale with the closest **logit** of the propensity score, $\log(e/(1-e))$, without replacement, within a **caliper** (maximum allowed distance) of 0.2 standard deviations of the logit. Treated sales with no control inside the caliper are dropped and reported (they are homes with no comparable non-assumable sale).

**Step 3: check balance.** For each covariate, the **standardized mean difference** is

$$\text{SMD} = \frac{\bar z_{\text{treated}} - \bar z_{\text{control}}}{\sqrt{(s^2_{\text{treated}} + s^2_{\text{control}})/2}},$$

with $s^2$ the group variances. The rule of thumb is $|\text{SMD}| < 0.1$ after matching. Balance is the test of the matching, not the model's fit; a propensity model can predict treatment well and still leave covariates unbalanced.

**Step 4: estimate the premium.** The simple estimate is the average matched-pair price difference. Better is **regression adjustment on the matched sample**: regress log price on treatment plus the covariates, using only matched sales,

$$\log Y_i = \mathbf{z}_i'\boldsymbol{\beta} + \Pi^{\%} D_i + e_i.$$

The coefficient $\Pi^{\%}$ is the premium in log points (about percent). This is **doubly robust**: it is consistent if either the matching or the regression is right, and it cleans up any residual imbalance the matching left.

**Step 5: capture ratio.**

$$\chi = \frac{\Pi}{\overline{V^{\text{loan}}}_{\text{treated}}},$$

the dollar premium divided by the average loan value (from 4.3) among the treated homes. If $\chi = 1$, the market pays full value; if $\chi = 0.25$, sellers capture a quarter and buyers keep the rest.

### Worked example

**Data.** Sales from January 2024 to September 2026 in the Virginia Piedmont region around Charlottesville: 16,000 sales, of which 283 were sold by assumption (1.8%).

**Propensity model.** The strongest predictor is the seller's purchase timing (coefficient 2.34 on "seller bought 2019–2022," odds multiplied by about 10). Smaller homes are more likely to be treated (−1.36 on log square feet). The treated sales' propensity scores range from about 0.5% to 13% (5th to 95th percentile), so every treated sale has many plausible comparisons; all 283 found a match within the caliper of 0.216.

**Balance.**

| Covariate | Treated mean | All controls mean | SMD before | Matched controls mean | SMD after |
|---|---|---|---|---|---|
| Square feet | 1,770 | 2,014 | −0.39 | 1,837 | −0.11 |
| House age (years) | 35.9 | 30.3 | +0.25 | 37.8 | −0.07 |
| Bedrooms | 3.74 | 4.04 | −0.28 | 3.88 | −0.13 |
| Miles to downtown | 8.50 | 7.99 | +0.09 | 8.26 | +0.04 |
| Sale quarter (index) | 5.43 | 5.01 | +0.13 | 5.33 | +0.03 |
| Seller bought 2019–22 | 0.72 | 0.21 | +1.22 | 0.74 | −0.02 |

Matching fixed the big imbalance (seller timing, from 1.22 to −0.02) but left size and bedrooms just over 0.1: the matched controls are still slightly larger. That is exactly the case regression adjustment exists for.

**Three estimates.**

| Estimate | Premium | Standard error |
|---|---|---|
| Raw difference in means (treated − all controls) | **−\$20,280** | |
| Matched-pair difference | +\$6,234 | \$5,142 |
| Regression-adjusted on the matched sample | **+3.80%** ≈ **+\$12,567** at the treated mean price of \$330,686 | 0.93 points |

The raw comparison says assumable homes sell for \$20,000 *less*, which is the cheaper-house effect, not the loan. Matching flips the sign but is noisy and still biased down by the size imbalance. The regression-adjusted estimate is tight: a 95% interval of 1.98% to 5.62%.

**Capture ratio.** The average 4.3 value among the 283 treated loans is \$45,916, so

$$\chi = \frac{12{,}567}{45{,}916} = 0.274,$$

with a 95% range of roughly 0.14 to 0.40. **The market pays about 27 cents per dollar of realistic loan value.** Measured against the naive value, the capture would look even lower (around 15%), which is one reason the naive figure is misleading for pricing advice.

**Applied to House H.** With $V^{\text{loan}} = \$51{,}038$, the market-typical premium is $0.274 \times 51{,}038 \approx \$14{,}000$ over the AVM value of \$415,000.

### Output

The premium $\Pi$ (percent and dollars, with interval), the capture ratio $\chi$ (overall and by segment: FHA vs. VA, price tier, county), the balance table, and the count of treated sales that could not be matched.

### Hand-off

$\chi$ and $\Pi$ go to the report (is the premium justified? how much more could sellers ask?) and to 4.5, where the agreed price sets the equity gap. Segment-level capture ratios feed the geographic "capture gap" map.

### Validation

Balance (every $|\text{SMD}| < 0.1$ after adjustment), a placebo test (estimate the "premium" for sellers with *non-assumable* loans bought in 2019–2022; it should be about zero, otherwise seller timing is proxying for something else), and sensitivity analysis for unmeasured confounding (how strong a hidden factor would have to be to erase the premium, Rosenbaum bounds). The main threat is condition: assumable sellers may be motivated sellers with less-updated homes. Listing-photo or remarks-based condition scores, where available, belong in $\mathbf{z}$.

---

## Step 4.5: Deal feasibility (effective rate + competing-risks time to close)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Sale price | $Q$ | Negotiated; informed by AVM (3.1) and premium (4.4) |
| Balance at closing | $B^{\text{close}}$ | 4.2 |
| Buyer's cash for the gap | $K$ | Buyer |
| Second-lien amount, rate, term | $L_2$, $c_2$, $n_2$ | Second-lien lender quotes |
| Fresh-loan alternative | $m_0$, $\mu^{\text{F}}$ | Same as 4.3 |
| Fees: assumption processing, second-lien closing, fresh-loan closing, FHA upfront MIP | | Servicer, lenders, FHA schedule |
| Assumption application histories: application date, close or fall-through date, servicer, loan type, equity gap, second lien | $\tau_j$, $\delta_j$, $\mathbf{w}_j$ | Servicer pipeline logs from client servicers; MLS listings marketed as assumable, joined to 4.1 outcomes |
| Loan value machinery | | 4.3 |

**Terms.** A **closed-end second** is a fixed-rate, fully amortizing second mortgage. The **upfront MIP** (UFMIP) is a one-time FHA insurance charge on a new loan, 1.75% of the loan amount, usually added to the balance; an assumption does not trigger it. **Fall-through** is an assumption application that ends without closing (credit denial, the second lien falls apart, the buyer walks away after delays).

### Process, part 1: the buyer's financing

**Equity gap and how it is funded.**

$$\text{EG} = Q - B^{\text{close}} = K + L_2.$$

**Blended rate.** The quick version weights each piece's note rate by its balance:

$$\bar c = \frac{B^{\text{close}}\, c + L_2\, c_2}{B^{\text{close}} + L_2}.$$

**Effective rate.** The quick version understates the cost, because the two pieces pay down at different speeds: the low-rate loan has a short remaining term and amortizes fast, while a 30-year second lien barely amortizes, so over time the expensive piece becomes a larger share of the debt. The correct measure is the **internal rate of return** (IRR) of the combined payments: the annual rate $c^{\text{eff}}$ such that

$$B^{\text{close}} + L_2 = \sum_{t=1}^{360} \frac{\text{PMT}^{\text{A}}_t + \mu^{\text{A}} B_{t-1}/12 + \text{PMT}_2}{\big(1 + c^{\text{eff}}/12\big)^{t}},$$

with $\text{PMT}^{\text{A}}_t$ the assumed loan's payment (zero after it is paid off) and $\text{PMT}_2$ the second lien's. Including the MIP term gives the **all-in** rate, comparable with the fresh loan's rate plus its MIP.

**Net value to the buyer.** The gross benefit is $V^{\text{loan}}$ from 4.3. Against it:

$$\text{NV} = V^{\text{loan}} - \underbrace{(Q - Q^{\text{AVM}})}_{\text{premium paid}} - \underbrace{V^{\text{2nd}}}_{\text{second-lien penalty}} + \underbrace{\Delta^{\text{upfront}}}_{\text{fees avoided}}.$$

The **second-lien penalty** $V^{\text{2nd}}$ is valued with the same machinery as 4.3 (same paths, same survival, same discounting), applied to the second lien's balance with "savings" $\big((c^{\text{F}}_t + \mu^{\text{F}}) - c_{2,t}\big) B_{2,t-1}/12$, which is negative because the second lien costs more than a fresh loan. On each path the buyer also refinances the second lien when rates fall far enough. The **upfront difference** is the fresh loan's UFMIP and closing costs minus the assumption fee and second-lien closing costs.

### Process, part 2: will it close, and when? (competing-risks survival)

Let $\tau$ be days since the assumption application. Each application ends in one of three ways: it **closes**, it **falls through**, or it is still **pending** when the data are pulled (**censored**: we know only that it lasted at least that long).

**Kaplan–Meier.** Ignoring covariates and treating anything other than closing as censoring, the probability of not yet having closed by $\tau$ is

$$\hat S(\tau) = \prod_{\tau_{(i)} \le \tau} \Big(1 - \frac{d_i}{n_i}\Big),$$

where $\tau_{(i)}$ are the distinct closing times, $d_i$ the number closing at that time, and $n_i$ the number still at risk (neither closed nor censored) just before it. Censored applications count in $n_i$ until they leave, which is how KM uses partial information instead of throwing it away.

**The competing-risks correction.** Treating fall-throughs as censored is wrong for the question "what is the chance this deal closes?" Censoring assumes the application *could still close later*; a fall-through never will. $1 - \hat S$ then estimates the closing probability in a world without fall-through and badly overstates it. The right object is the **cumulative incidence function** (Aalen–Johansen estimator):

$$F_{\text{close}}(\tau) = \sum_{\tau_{(i)} \le \tau} \hat S_{\text{any}}\big(\tau_{(i)}^{-}\big)\, \hat\lambda_{\text{close}}\big(\tau_{(i)}\big),$$

where $\hat S_{\text{any}}(\tau^{-})$ is the probability of still being pending (neither closed nor fallen through) just before $\tau$, and $\hat\lambda_{\text{close}}$ the closing hazard at that time. In words: to close on day $\tau$ you must still be alive on day $\tau$, and then close.

**Cox model.** Covariates enter through **cause-specific proportional hazards**, one model for closing and one for fall-through:

$$\lambda_{\text{close}}(\tau \mid \mathbf{w}) = \lambda_{0,\text{close}}(\tau)\, \exp\!\big(\mathbf{w}'\boldsymbol{\beta}_{\text{close}}\big), \qquad \lambda_{\text{fall}}(\tau \mid \mathbf{w}) = \lambda_{0,\text{fall}}(\tau)\, \exp\!\big(\mathbf{w}'\boldsymbol{\beta}_{\text{fall}}\big).$$

$\lambda_0$ is the **baseline hazard** (left unspecified, estimated nonparametrically by the Breslow estimator), and $e^{\beta}$ is a **hazard ratio**: a hazard ratio of 0.72 means 28% fewer closings per day at every point in the process. The coefficients are estimated by maximizing the **partial likelihood**, which compares each event with everyone still at risk at that moment and never needs $\lambda_0$:

$$\mathcal{L}(\boldsymbol{\beta}) = \prod_{\text{events } i} \frac{\exp(\mathbf{w}_i'\boldsymbol{\beta})}{\sum_{j \in \text{risk set}(\tau_i)} \exp(\mathbf{w}_j'\boldsymbol{\beta})}.$$

Covariates $\mathbf{w}$: servicer (from 4.1), loan type, equity gap as a share of price, and whether a second lien is needed. For a specific deal, plug its covariates into both models, then combine them in the cumulative incidence formula.

### Worked example: financing

The buyer and seller agree on **$Q = \$427{,}000$**: a \$12,000 premium over the AVM's \$415,000 (capture 23.5%, a bit under the market's 27%).

$$\text{EG} = 427{,}000 - 282{,}812 = \$144{,}188 \;\; (33.8\% \text{ of price}).$$

The buyer brings $K = \$44{,}188$ in cash and takes a 30-year closed-end second of $L_2 = \$100{,}000$ at $c_2 = 8.50\%$: $\text{PMT}_2 = \$768.91$.

**Rates.**

| Measure | Value |
|---|---|
| Blended note rate $\bar c = (282{,}812 \times 3.00\% + 100{,}000 \times 8.50\%)/382{,}812$ | 4.44% |
| Effective rate (IRR of note payments) | 4.79% |
| All-in effective rate (IRR including the 0.85% MIP) | **5.35%** |
| Fresh FHA alternative, all-in (7.00% + 0.55% MIP) | **7.55%** |

The IRR sits 0.35 points above the blended rate because the 3% loan is gone after 285 months while the 8.5% second runs for 360.

**Monthly payment.** The alternative: buy at \$415,000 with the same cash, borrowing \$370,812 fresh.

| | Assumption | Fresh FHA |
|---|---|---|
| First mortgage P&I | \$1,391.29 | \$2,467.02 |
| MIP (first month) | \$200.33 | \$169.96 |
| Second lien P&I | \$768.91 | |
| **Total** | **\$2,360.53** | **\$2,636.98** |

The buyer saves **\$276 a month** even after paying a \$12,000 premium and borrowing \$100,000 at 8.5%.

**Net value.**

| Component | Amount |
|---|---|
| Loan value $V^{\text{loan}}$ (4.3) | +\$51,038 |
| Premium paid | −\$12,000 |
| Second-lien penalty $V^{\text{2nd}}$ (same 20,000 paths; 8.5% vs. 7.55%, refinanced when rates fall) | −\$6,789 |
| Upfront: UFMIP avoided (1.75% × \$370,812 = \$6,489) + fresh closing costs avoided (\$4,000) − assumption fee (\$1,000) − second-lien closing (\$1,500) | +\$7,989 |
| **Net value to buyer** | **+\$40,238** |

The deal is clearly good for the buyer, which also says the seller left money on the table: a buyer could rationally pay up to roughly \$40,000 more (somewhat less once the larger second lien needed to fund it is costed), but the market only clears near 27% capture because so few buyers can fund the gap or wait out the servicer.

(Simplification noted: we used FHA's 0.55% MIP for the fresh alternative; at this buyer's 89% loan-to-value FHA would charge 0.50% for 11 years, which changes the net value by roughly \$1,000.)

### Worked example: time to close

**Kaplan–Meier by hand.** Eight recent applications at one servicer, days from application: closed at 48, 63, 77, 77, 95, 128; pending at 70 and 110 (censored).

| Day | At risk $n_i$ | Closed $d_i$ | $1 - d_i/n_i$ | $\hat S(\tau)$ |
|---|---|---|---|---|
| 48 | 8 | 1 | 7/8 | 0.875 |
| 63 | 7 | 1 | 6/7 | 0.750 |
| 70 | (one censored; at risk drops to 5) | | | 0.750 |
| 77 | 5 | 2 | 3/5 | 0.450 |
| 95 | 3 | 1 | 2/3 | 0.300 |
| 110 | (one censored) | | | 0.300 |
| 128 | 1 | 1 | 0 | 0 |

The median time to close is the first day $\hat S \le 0.5$: **77 days**. Simply averaging the six closed applications (81 days) drops the information that two others were still pending at 70 and 110 days.

**Cox fit on the full history.** 2,400 applications from 2023 to 2026: 58% closed, 33% fell through, 9% still pending. Median days to close among those that closed: 87. Fitted hazard ratios (reference: Servicer A, FHA, zero gap, no second lien):

| Covariate | HR for closing | HR for fall-through |
|---|---|---|
| Servicer B | 0.72 | 1.15 |
| Servicer C | 0.44 | 1.33 |
| VA loan | 0.78 | 1.03 |
| Equity gap, per 10 points of price | 0.90 | 1.27 |
| Second lien needed | 0.76 | 1.24 |

Every obstacle both slows closing and raises fall-through. The servicer is the largest single factor: an application at Servicer C closes at less than half the daily rate of one at Servicer A.

**House H's deal.** Servicer B (Loan L's issuer from 4.1), FHA, equity gap 33.8%, second lien. Linear predictors:

$$\exp(\mathbf{w}'\boldsymbol{\beta}_{\text{close}}) = 0.72 \times 0.90^{3.38} \times 0.76 \approx 0.38, \qquad \exp(\mathbf{w}'\boldsymbol{\beta}_{\text{fall}}) = 1.15 \times 1.27^{3.38} \times 1.24 \approx 3.2.$$

This deal closes at 38% of the reference rate and falls through at more than three times it. Cumulative incidence:

| Days from application | Closed | Fell through | Still pending |
|---|---|---|---|
| 60 | 13.9% | 17.7% | 68.4% |
| 90 | 30.9% | 25.1% | 44.1% |
| 120 | **45.2%** | 30.6% | 24.1% |
| 150 | 56.5% | 33.4% | 10.1% |
| 180 | **62.2%** | 34.4% | 3.3% |

The naive $1 - \hat S$ that treats fall-throughs as censored says 92% by day 180. The honest answer is 62%, with a one-in-three chance the deal dies. For comparison, the same deal at Servicer A would close by day 120 with probability 57%, and a simple deal at Servicer A (20% gap, no second lien) with 76%.

**Implication for the contract.** A standard 45-day closing window is hopeless (7% chance). A 120-day window gives even odds. The listing should state the servicer's typical timeline up front, and the seller should start the servicer's assumption package at listing rather than at contract.

### Output

Per deal: equity gap, cash and second-lien sizing, blended, effective and all-in rates, monthly payment comparison, net value to the buyer, and the closing / fall-through / pending curves. Per servicer: the hazard ratios, which double as a servicer scorecard.

### Hand-off

All of it goes to the report. The servicer hazard ratios also go to lender and servicer clients as a benchmark of their own assumption operations.

### Validation

Out-of-time: fit on applications through 2025, predict 2026 cumulative incidence by servicer and compare with what happened. Check proportional hazards with Schoenfeld residuals (if the servicer effect fades over time, stratify the baseline by servicer instead). Recompute the effective rate against a lender's own disclosure (APR) for a few real deals.

---

## The report

### Seller and listing agent card (House H)

- **Your loan:** FHA, 3.00%, about \$282,800 at a March 2027 closing (confirmed against Ginnie Mae's September balance; match confidence 99%). Assumable with the servicer's credit review.
- **What it is worth to a buyer:** about **\$51,000** realistically (range \$42,000 to \$57,000 across rate scenarios). Ignore the \$90,000 "payment difference to maturity" figure; most buyers will move or refinance long before 2050.
- **What the market pays:** sellers in this region capture about 27% of that value, about **\$14,000** over a comparable home. Suggested list price: **\$429,000 to \$432,000** (AVM \$415,000), marketed with the buyer's monthly payment, not just the rate.
- **The obstacle:** a buyer needs about **\$144,000** to cover your equity, so plan for buyers with large cash or a second-lien lender lined up. Your servicer is in the slower half; start the assumption package at listing, and write a contract window of at least 120 days.

### Buyer card (House H at \$427,000)

- **Cash needed:** \$44,188 plus closing costs, with a \$100,000 second lien at 8.50%.
- **Effective rate:** 5.35% all-in versus 7.55% on a fresh FHA loan. **Monthly payment \$2,361 versus \$2,637.**
- **Net value of the deal:** about **\$40,000** after the premium, the cost of the second lien, and fees.
- **Timing risk:** 45% chance of closing within 120 days, 62% within 180, and about a one-in-three chance the assumption falls through. Keep a fresh-loan pre-approval as a backup.

### Market table (illustrative, Charlottesville area)

| Locality | Active sub-4% FHA/VA loans linked to parcels | Mean realistic loan value | Sales by assumption, 2024–26 | Listing capture | Price capture $\chi$ |
|---|---|---|---|---|---|
| Albemarle County | 2,140 | \$47,800 | 61 | 9% | 0.29 |
| Charlottesville city | 690 | \$44,100 | 14 | 6% | 0.22 |
| Greene County | 820 | \$43,500 | 22 | 11% | 0.31 |
| Fluvanna County | 1,050 | \$41,900 | 19 | 8% | 0.25 |
| Louisa County | 1,210 | \$39,600 | 17 | 7% | 0.24 |

**Listing capture** is the share of sales of homes carrying an eligible low-rate loan that actually closed by assumption; **price capture** is $\chi$ from 4.4. Low listing capture means sellers aren't using the asset at all (a marketing gap for brokerages); low price capture means they are using it but underpricing it (a pricing gap). Charlottesville city is low on both.

### For lenders, servicers, and investors

- **Servicers:** the time-to-close hazard ratios rank assumption operations; Servicer C's closing rate is 44% of the best performer's. Faster processing raises both closing probability and the borrower's sale price.
- **Second-lien lenders:** the equity gap is the binding constraint. In this market most assumable homes need a \$100,000+ second; the net-value calculation shows buyers can afford 8.5% on it.
- **Investors in Ginnie Mae pools:** a loan that is assumed stays in the pool (no prepayment) where an ordinary sale would have paid it off at par. For a deeply discounted 3% pool, rising assumption rates *slow* prepayments, which lowers the pool's value. The assumption counts by locality feed the turnover model in Service 5.

---

## Symbol reference

| Symbol | Meaning | Step |
|---|---|---|
| $i$ | Property (house) | all |
| $g$, $d$ | Ginnie Mae loan record; recorder document | 4.1 |
| $\mathbf{z}_d$, $\boldsymbol{\omega}_k$, $q_{d,k}$ | Document text features; classifier coefficients for type $k$; predicted type probability | 4.1 |
| $f = 1, \dots, F$ | Comparison field | 4.1 |
| $\gamma^f_{gd}$, $\boldsymbol{\gamma}_{gd}$ | Field agreement indicator; comparison vector | 4.1 |
| $M$, $U$ | Sets of true matches and non-matches | 4.1 |
| $m_f$, $u_f$ | Agreement probability among matches; among non-matches | 4.1 |
| $w_f$, $W_{gd}$ | Field weight; total weight (log likelihood ratio) | 4.1 |
| $\pi$, $P_{gd}$ | Match prevalence in block; posterior match probability | 4.1 |
| $t_U$, $t_L$ | Upper and lower decision thresholds | 4.1 |
| $B_0$, $c$, $n$, $\rho$ | Original amount; annual note rate; original term (months); monthly rate $c/12$ | 4.2 |
| PMT | Monthly principal and interest payment | 4.2 |
| $B_k$, $\tilde B_{k_0}$ | Scheduled balance after $k$ payments; reported balance after $k_0$ payments | 4.2 |
| $X$ | Cumulative curtailment (extra principal paid) | 4.2 |
| $B^{\text{close}}$, $B_t$, $T$ | Balance at closing; balance $t$ months after closing; remaining months | 4.2, 4.3 |
| $r_t$, $\theta(t)$, $a$, $\sigma$, $W_t$ | Short rate; calibrated drift; mean reversion; volatility; Brownian motion | 4.3 |
| $x_t$, $\varphi(t)$, $\Delta$ | Hull-White factor; deterministic curve part; time step | 4.3 |
| $m_{n,t}$, $f^{m}_t$, $b_{10}$ | Fresh-loan rate on path $n$; forward-implied fresh rate; 10-year loading | 4.3 |
| $c^{\text{F}}_t$, $c^{\text{A}}_t$ | Rate paid by fresh-loan borrower; by assumer | 4.3 |
| $\mu^{\text{F}}$, $\mu^{\text{A}}$ | Annual MIP rate, fresh loan; assumed loan | 4.3, 4.5 |
| $\kappa$, $k^{\text{R}}$ | Refinancing threshold; refinancing cost share | 4.3 |
| $s_t$ | Monthly savings from holding the assumed loan | 4.3 |
| $h_t$, $h^{\text{sale}}_t$, $h^{\text{def}}$, $h^{\text{oth}}$ | Exit hazard and its components | 4.3 |
| $S(t)$, $S_n(t)$ | Survival (still holding the loan); path-specific | 4.3 |
| $\eta_0$, $\theta_g$, $G_t$, $\text{ten}(t)$ | Baseline sale log odds; lock-in coefficient; lock-in gap; tenure multiplier | 4.3 |
| $\text{DF}_{n,t}$, $N$ | Discount factor; number of paths | 4.3 |
| $V^{\text{loan}}$ | Realistic value of the loan to a buyer | 4.3 |
| $D_i$, $Y_i$, $\mathbf{z}_i$ | Sold by assumption; sale price; characteristics | 4.4 |
| $e(\mathbf{z})$, $\boldsymbol{\alpha}$ | Propensity score; its coefficients | 4.4 |
| SMD | Standardized mean difference | 4.4 |
| $\Pi$, $\Pi^{\%}$ | Price premium (dollars, log points) | 4.4 |
| $\chi$ | Capture ratio: premium ÷ loan value | 4.4 |
| $Q$, $Q^{\text{AVM}}$ | Sale price; AVM market value | 4.5 |
| EG, $K$ | Equity gap; buyer cash | 4.5 |
| $L_2$, $c_2$, $\text{PMT}_2$ | Second-lien amount, rate, payment | 4.5 |
| $\bar c$, $c^{\text{eff}}$ | Blended note rate; effective rate (IRR) | 4.5 |
| $V^{\text{2nd}}$, $\Delta^{\text{upfront}}$, NV | Second-lien penalty; upfront fee difference; net value to buyer | 4.5 |
| $\tau$ | Days since assumption application | 4.5 |
| $\hat S(\tau)$, $n_i$, $d_i$ | Kaplan–Meier survival; number at risk; number of events | 4.5 |
| $F_{\text{close}}(\tau)$, $\hat S_{\text{any}}$ | Cumulative incidence of closing; probability still pending | 4.5 |
| $\lambda_0$, $\boldsymbol{\beta}_{\text{close}}$, $\boldsymbol{\beta}_{\text{fall}}$, $\mathbf{w}$ | Baseline hazard; Cox coefficients by outcome; deal covariates | 4.5 |
