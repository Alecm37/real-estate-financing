# Service 5: MBS Turnover Forecasting, explained as a pipeline

*Market backdrop: mid-October 2026. All pool-level and loan-level numbers below come from one worked example run end to end on a simulated pool, with a simulated rate model; they are illustrative, not quotes of any real pool, dealer or price.*

---

## 0. What we are forecasting, and why it needs five models

### The object being forecast

When a bank or mortgage company makes a home loan that meets Fannie Mae's or Freddie Mac's standards (a **conforming** loan), it usually does not keep it. It sells it to one of these two government-sponsored enterprises (**GSEs**), which bundle thousands of similar loans into a **pool** and issue a security against it: a **mortgage-backed security (MBS)**. The standard form is a **pass-through**: every month, whatever the borrowers in the pool pay is passed through to the investors, minus fees. Since 2019, Fannie and Freddie issue a common security called **UMBS** (Uniform MBS). The GSE guarantees the investor against credit losses, so an agency MBS investor never loses principal to default. What the investor does not know is **when** the principal comes back.

Some vocabulary we will use throughout:

- **Note rate** $c_i$: the interest rate borrower $i$ pays on the loan. **WAC** (weighted-average coupon) is the balance-weighted average note rate in a pool.
- **Pass-through coupon** $c^{\text{pt}}$: the rate the investor receives on the pool's balance. It is below the WAC because a slice goes to the **servicer** (the company that collects payments) and to the GSE as its **guarantee fee**. Pool T below has WAC 2.86% and coupon 2.0%.
- **Scheduled principal**: the principal part of each borrower's normal monthly payment, set by the amortization schedule.
- **Prepayment**: any principal paid *beyond* the schedule. A **full payoff** happens when the borrower sells the house (the loan must be repaid at closing), refinances (a new loan pays off the old one), or defaults (the GSE buys the loan out of the pool at par, which the investor sees as a prepayment). A **curtailment** is a partial prepayment: the borrower sends extra principal but keeps the loan.
- **Pool factor**: current balance ÷ original balance.
- **SMM** (single monthly mortality): the share of the balance outstanding after scheduled principal that is prepaid in a month. **CPR** (conditional prepayment rate) is its annualized version, $\text{CPR} = 1 - (1 - \text{SMM})^{12}$. A CPR of 5% means that at this month's pace, 5% of the balance would prepay over a year.
- **Price** is quoted per \$100 of current balance, in points and 32nds: 77-28 means $77 + 28/32 = 77.875$.
- **Discount** pool: one priced below 100, because its coupon is below what new mortgages pay. **Premium** pool: priced above 100.
- **TBA** ("to be announced"): the main trading market for agency MBS. A buyer agrees on coupon, issuer type and settlement month, and the seller delivers *any* eligible pools. Sellers naturally deliver the least valuable eligible pools, so TBA prices reflect the cheapest pools in the cohort.
- **Specified pool**: a pool traded by its own identity, because it has some feature (all loans in one state, small loan balances) that makes its prepayments different. The extra price over TBA is the **pay-up**.

The quantity this service forecasts, for each pool $p$, is the **path of monthly prepayment rates** (an SMM for every future month, together with its distribution under rate uncertainty), broken down by **cause**: how much comes from home sales (**turnover**), how much from refinancing, how much from default buyouts and curtailments. It then turns that forecast into **value**: what the pool is worth, and whether its market price is rich or cheap.

### Why it is worth forecasting now

The backdrop makes turnover the only part of prepayment that matters for most of the market:

- The 30-year fixed rate is near 7.3% (Freddie Mac PMMS). About half of outstanding mortgages carry rates below 4%, and only about 22% are above 6%. For the low-rate half, refinancing is pointless: no one trades a 2.75% loan for a 7.3% loan. This is the **lock-in effect**.
- So the principal of a deeply discounted pool (2.0s, 2.5s, 3.0s from 2020 and 2021) comes back in only three ways: scheduled amortization, which is slow at low note rates; the occasional default buyout; and **turnover**, meaning the borrower sells the house and moves. Turnover is the only one of these that varies a lot from pool to pool.
- For a discount pool, faster principal return is *good* for the investor: every dollar returned early is a dollar bought at about 78 and repaid at 100. In the worked example below, each extra 1 CPR of sustained speed is worth roughly **1½ points** of price on Pool T. On a \$100 million position that is \$1.5 million, which is why investors pay up for pools they believe will turn over faster.
- Turnover itself is depressed and uneven. Home prices are roughly flat nominally and falling in real terms; existing-home supply is about 4.9 months; job and migration flows differ sharply by region. A national average speed assumption misprices regional pools in both directions.
- Delinquencies are about 4.37% and rising, concentrated in FHA loans. FHA and VA loans are pooled by Ginnie Mae, not the GSEs, but rising stress also lifts default buyouts in conventional pools, a small third channel that has to be separated from turnover.

### Why five models instead of one

The obvious approach is to regress each pool's past CPR on rates and pool characteristics. It fails for this problem for three reasons. First, **causes are mixed**: a payoff in the data is just a loan that disappeared, and a sale and a refinance respond to opposite things (rates up suppresses refinancing *and* sales, but through different mechanisms and with different sizes). Second, **the useful variation is local and recent**: turnover depends on the regional housing market this month, which pool histories reveal only months later. Third, **the investor needs value, not speed**: a CPR forecast must be pushed through many interest-rate paths and a discounting step before it can say "buy" or "sell."

So we build it in five stages:

| Step | Model | Produces |
|---|---|---|
| 5.1 | Anchor labeling + two-component mixture (EM) | For every historical payoff, the probability it was a sale vs. a refinance |
| 5.2 | Competing-risks hazard model (multinomial logit, then monotone gradient boosting) | For every loan and future month, the probability of sale, refinance, default |
| 5.3 | Dynamic factor model + Kalman filter | A real-time regional turnover index, fed into 5.2 |
| 5.4 | Balance-weighted aggregation + credibility weighting | Pool SMM and CPR paths, adjusted for the pool's own track record |
| 5.5 | Hull-White rate paths + cash-flow engine + OAS | Value, option-adjusted spread, implied speeds, rich/cheap ranking |

### The actual flow of numbers

```
Loan-level performance data ──► 5.1 payoff labels (sale / refi weights) ─┐
                                                                           ├─► 5.2 hazard model: h^S, h^R, h^D per loan-month
Regional indicators ──► 5.3 Kalman filter ──► turnover index f_t ─────────┤
AVM (3.1) ──► current LTV ─────────────────────────────────────────────────┘
                                                                           │
                    pool's own recent payoffs ──► 5.4 credibility multiplier μ_p
                                                                           │
                                                                           ▼
           5.4 aggregation: pool SMM_t and CPR_t  (as a function of the rate path)
                                                                           │
   Hull-White rate paths ──► mortgage rate path m_t ──► 5.5 cash flows, OAS, implied speed
                                                                           │
                                                                           ▼
                   CPR curves, turnover vs. refi attribution, rich/cheap ranking ──► report
```

Three links deserve emphasis. First, 5.1 does not forecast anything; it **creates the training labels** for 5.2, because the data never say why a loan paid off. Every downstream number inherits its errors, so it is validated separately. Second, the turnover index from 5.3 enters 5.2 as an ordinary feature, but its *forecast* fades toward the long-run average over the projection, so local information dominates the next year and matters little after five. Third, 5.2's hazards are a **function of the mortgage rate path**, not numbers: 5.5 re-evaluates them on every simulated path, because a rate rally both raises refinancing and loosens lock-in.

### Timeline used in the worked example

- **Today:** mid-October 2026.
- **Latest loan-level data:** September 2026 activity, disclosed in early October (the GSEs release monthly loan-level and pool-level files).
- **Latest regional indicators:** September 2026 for pending sales, new listings and job flows; migration data lag and are not yet in for September.
- **Valuation date:** start of November 2026 (November settlement, simplified). Projection month 1 is **November 2026**.
- **Payment delay:** a UMBS investor receives a month's principal and interest on the 25th of the *following* month (a 55-day stated delay), so November's cash flow arrives December 25, about 1.83 months after the valuation date.

### The running example: Pool T and Loan L

**Pool T** is a 30-year UMBS 2.0% specified pool, 100% Texas loans, issued in fall 2021.

| Characteristic | Value |
|---|---|
| Loans currently in pool | 600 |
| Current balance | \$175.9 million (factor about 0.67) |
| WAC / pass-through coupon | 2.86% / 2.00% |
| WALA (weighted-average loan age) | 60 months |
| Average loan balance | \$293,000 |
| Weighted-average credit score (FICO) | 757 |
| Weighted-average current LTV | 67% |
| Market: TBA UMBS 2.0 price | 77-28 (77.875) |
| Market: Pool T price | 78-00, a pay-up of 4/32 |

**Loan L**, one loan inside Pool T, carries the arithmetic through 5.2 and 5.4:

- Originated October 2021 in the Dallas–Fort Worth area, \$320,000, 30-year fixed at **2.75%**, monthly principal and interest **\$1,306.37**.
- Purchase price \$400,000 (LTV 80% at origination, where **LTV**, loan-to-value, is loan balance ÷ property value). The AVM (3.1) currently values the home at **\$414,000**.
- Credit score 765. After 60 payments (end of October 2026) the balance is **\$283,186.87**.

The punchlines we will derive: Pool T is projected at about **4.9 CPR** over the next year, roughly **80%** of its prepaid principal will come from home sales, and at 78-00 it is worth a little more than it costs: our model says the Texas pool deserves a pay-up of about 6/32 against the 4/32 the market charges. That edge is small next to the model's own uncertainty, and the report says so.

---

## Step 5.1: Payoff decomposition (anchor labeling + mixture/EM)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Loan-level monthly performance: balance, status, payoff month, zero-balance code | | Fannie Mae Single-Family Loan Performance Data; Freddie Mac Single-Family Loan-Level Dataset |
| Note rate of each loan | $c_i$ | Same (origination file) |
| Market mortgage rate by month | $m_t$ | Freddie Mac PMMS |
| Rate incentive at payoff | $G_e = c_{i(e)} - m_{t(e)-1}$ | Computed |
| Payoff features: calendar quarter of payoff, recent rate drop | $\text{qtr}_e$, $\text{drop}_e$ | Computed |
| Validation subsample: payoffs linked to a recorded deed | | County recorder records joined via record linkage (4.1), or MSA-level sales counts |

Here $e$ indexes **payoff events** (one per loan that paid off in full), $i(e)$ is the loan and $t(e)$ the month of payoff.

**Terms.** The **zero-balance code** in the agency data records why a loan left the pool: code 01 is "prepaid or matured" (a voluntary payoff), while the others mark credit events (short sale, foreclosure sale, real-estate-owned disposition) or repurchases. Credit events are the default outcome $D$ and are labeled directly. The trouble is code 01: it lumps together **sales** and **refinances**, and nothing in the anonymized agency data says which. The **rate incentive** $G$ is note rate minus market rate: positive means the borrower could save by refinancing (the loan is **in the money**), negative means refinancing would raise their rate (**out of the money**). We measure it with the market rate one month before payoff, $m_{t-1}$, because a loan that closes this month was applied for about a month earlier.

### Process

**Stage 1: anchors.** Some payoffs are almost certainly sales. A borrower whose loan is at 3% when the market is at 7% does not refinance into a higher rate to stay in the same house; if that loan pays off, the house was almost certainly sold. So we define the **anchor set**

$$\mathcal{A} = \{ e : G_e \le -1.5 \text{ percentage points} \},$$

and label every anchor as a sale. The one real contaminant is **cash-out refinancing** (a new, larger loan taken to pull equity out of the house), which some borrowers do even out of the money. On the validation subsample we measure the share of anchors with no recorded sale within 60 days of the payoff, call it $\varepsilon$, and give anchors a sale weight of $1-\varepsilon$ rather than 1. In the 2023 to 2026 data, when almost every loan was out of the money, the anchor set is huge, which is the lucky part of today's market for this method.

**Stage 2: the sale profile.** From the anchors we learn what a sale-driven payoff *looks like* in features that do not depend on the incentive. Two features carry most of the signal:

- **Calendar quarter** $\text{qtr}_e$: home sales close disproportionately in spring and summer (families move between school years); refinancing has no calendar.
- **Recent rate drop** $\text{drop}_e = 1$ if the market rate fell by at least 25 basis points (0.25 percentage point) in the three months before the payoff. Refinances cluster right after rate drops; sales barely react.

Denote the sale profile $f_S(\text{qtr}, \text{drop})$, estimated as frequencies in $\mathcal{A}$.

**The identifying assumption, stated plainly:** *a sale looks the same whether or not the loan happened to be in the money.* That is, the seasonality and rate-drop pattern of sales among in-the-money loans equals that among anchors. It is what lets anchors teach us about ambiguous payoffs, and it is checked on the validation subsample.

**Stage 3: the mixture.** For payoffs in an ambiguous incentive band (say $+0.5 \le G \le +1.5$), each payoff is a sale with unknown probability $\pi$ and a refinance otherwise. Its features then follow the two-component mixture

$$\Pr(\text{qtr}_e, \text{drop}_e) = \pi\, f_S(\text{qtr}_e, \text{drop}_e) + (1-\pi)\, f_R(\text{qtr}_e, \text{drop}_e).$$

$f_S$ is fixed from the anchors; $\pi$ and the refinance profile $f_R$ are unknown. To keep $f_R$ estimable we assume its two features are independent given that the payoff is a refinance (a **naive Bayes** factorization), $f_R(\text{qtr}, \text{drop}) = f_R^{\text{q}}(\text{qtr})\, f_R^{\text{d}}(\text{drop})$, which leaves 3 + 1 free numbers in $f_R$ plus $\pi$.

**Estimation by EM** (expectation-maximization), which alternates two steps until nothing changes:

- **E-step:** with current $\pi, f_R$, compute each payoff's posterior sale probability by Bayes' rule,
$$\omega_e = \frac{\pi\, f_S(\text{qtr}_e, \text{drop}_e)}{\pi\, f_S(\text{qtr}_e, \text{drop}_e) + (1-\pi)\, f_R(\text{qtr}_e, \text{drop}_e)}.$$
- **M-step:** re-estimate the unknowns as if the $\omega_e$ were real labels:
$$\pi^{\text{new}} = \frac{1}{n}\sum_e \omega_e, \qquad f_R^{\text{q}}(q) = \frac{\sum_e (1-\omega_e)\,\mathbb{1}\{\text{qtr}_e = q\}}{\sum_e (1-\omega_e)}, \qquad f_R^{\text{d}}(1) = \frac{\sum_e (1-\omega_e)\,\text{drop}_e}{\sum_e (1-\omega_e)},$$
where $n$ is the number of ambiguous payoffs and $\mathbb{1}\{\cdot\}$ is 1 when the condition holds and 0 otherwise.

Each EM iteration provably does not decrease the log-likelihood $\sum_e \log \Pr(\text{qtr}_e, \text{drop}_e)$, so it climbs to a maximum. In production, $\pi$ is allowed to vary smoothly with the incentive $G$, loan age and other covariates (a logistic regression inside the M-step), and the band runs over the full range of $G$. The worked example uses one band to keep the arithmetic visible.

### Worked example

**Anchors.** 4,000 payoffs with $G \le -1.5$. Their profile:

| | Q1 | Q2 | Q3 | Q4 |
|---|---|---|---|---|
| $f_S^{\text{q}}$ (share of sales in quarter) | 0.162 | 0.314 | 0.319 | 0.205 |

and $\Pr(\text{drop}=1 \mid \text{sale}) = 0.198$. So, for example, $f_S(\text{Q3}, \text{drop}=0) = 0.319 \times 0.802 = 0.2558$.

**Ambiguous band.** 2,000 payoffs with $+0.5 \le G \le +1.5$:

| Count | Q1 | Q2 | Q3 | Q4 |
|---|---|---|---|---|
| drop = 0 | 223 | 301 | 317 | 224 |
| drop = 1 | 198 | 261 | 233 | 243 |

Already visible: the drop = 1 row is far heavier than the anchors' 20% would predict, so many of these payoffs must be refinances.

**Iteration 1, E-step.** Start at $\pi = 0.5$ and a flat refinance profile, $f_R = 0.25 \times 0.5 = 0.125$ in every cell. For a Q3 payoff with no recent rate drop:

$$\omega = \frac{0.5 \times 0.2558}{0.5 \times 0.2558 + 0.5 \times 0.125} = 0.672.$$

For a Q1 payoff right after a rate drop, $f_S = 0.162 \times 0.198 = 0.0321$ and $\omega = 0.204$. The full posterior table after this E-step:

| $\omega$ | Q1 | Q2 | Q3 | Q4 |
|---|---|---|---|---|
| drop = 0 | 0.510 | 0.668 | 0.672 | 0.568 |
| drop = 1 | 0.204 | 0.332 | 0.336 | 0.245 |

**Iteration 1, M-step.** Summing $\omega$ over all 2,000 payoffs (cell count × cell posterior) gives 919.9, so $\pi^{\text{new}} = 919.9/2{,}000 = 0.460$. The refinance weight $\sum(1-\omega) = 1{,}080.1$ splits across quarters as 266.9, 274.2, 258.8, 280.2, so $f_R^{\text{q}} = (0.247, 0.254, 0.240, 0.259)$; and 670.1 of it sits in drop = 1 cells, so $f_R^{\text{d}}(1) = 0.620$. The log-likelihood rises from −4,198.7 to −4,145.8.

**Convergence.** Iterating, the estimates settle at

$$\hat\pi = 0.364, \qquad \hat f_R^{\text{q}} = (0.233, 0.266, 0.244, 0.256), \qquad \hat f_R^{\text{d}}(1) = 0.622,$$

with final posteriors

| $\omega$ (final) | Q1 | Q2 | Q3 | Q4 |
|---|---|---|---|---|
| drop = 0 | 0.458 | 0.589 | 0.614 | 0.493 |
| drop = 1 | 0.112 | 0.177 | 0.192 | 0.127 |

The model concluded that refinances are spread evenly through the year and that 62% of them follow a rate drop, against 20% of sales. That makes sense economically, and it was not assumed: only $f_S$ was fixed.

**How well is $\pi$ pinned down?** This example was simulated, and the true sale share was 0.35, so 0.364 is close. But the likelihood is **flat** in $\pi$: holding $\pi$ at 0.30 or 0.45 and re-optimizing everything else lowers the log-likelihood by only 0.14 and 0.36. (EM also needed about 2,000 iterations to settle, which is a symptom of the same flatness.) Two features and one band identify the sale share only loosely. In production, the strength comes from pooling across many incentive bands, letting $\pi$ follow a smooth curve in $G$ that must connect to $\pi \approx 1$ at the anchors, and adding features (loan age, local sales volume that month). This is a place to report a range, not a point.

### Output

For every voluntary payoff $e$ in the training history, a **soft label**: probability $\omega_e$ of sale and $1-\omega_e$ of refinance (anchors get $1-\varepsilon$ and $\varepsilon$; credit events are labeled default with probability 1). Also the estimated sale share by incentive band, a useful report output on its own ("in 2025, 36% of in-the-money payoffs were really moves").

### Hand-off

To 5.2, where each payoff month in the loan-month panel carries fractional outcomes ($\omega_e$ sale, $1-\omega_e$ refinance) instead of a single label.

### Validation

On the linked subsample (payoffs matched to a deed record), compare predicted $\omega$ with whether a sale was actually recorded. In the simulation, a random 400 ambiguous payoffs played that role:

| Posterior bin | Payoffs | Mean predicted $\omega$ | Actual sale share |
|---|---|---|---|
| below 0.3 | 214 | 0.155 | 0.121 |
| 0.3 to 0.6 | 129 | 0.520 | 0.512 |
| 0.6 and above | 57 | 0.614 | 0.561 |

Slightly over-labeling sales in this sample, consistent with $\hat\pi = 0.364$ vs. a true 0.35, and within sampling noise for these bin sizes. Where no linked subsample exists (Fannie and Freddie data carry no address, only a 3-digit zip and an MSA), validate in aggregate: the count of sale-labeled payoffs by MSA and month should track recorded home-sales counts in that MSA.

---

## Step 5.2: Competing-risks hazard model

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Loan-month panel: one row per loan per month alive | | Agency loan-level data |
| Outcome each month: survive, sale, refinance, default | $y^{k}_{i,t}$, $k \in \{0, S, R, D\}$ | Soft labels from 5.1 |
| Rate incentive and lock-in gap | $G_{i,t}$, $L_{i,t}$ | Note rate and PMMS |
| Loan age | $\text{age}_{i,t}$ | Months since first payment |
| Current LTV | $\text{CLTV}_{i,t}$ | Current balance ÷ AVM value (3.1), or origination value × FHFA HPI |
| Credit score, loan size, state | $\text{FICO}_i$, $b_{i,t}$, $\text{TX}_i$ | Origination file |
| Calendar month of the year | $\text{moy}(t)$ | |
| Local turnover index | $f_{j(i),t}$ | Step 5.3, for loan $i$'s region $j(i)$ |

### Why a competing-risks model

Each month a loan faces three ways to leave: sale, refinance, default. They **compete**: whichever happens first ends the loan, and the loan cannot then do the others. Modeling, say, the sale hazard alone and treating refinanced loans as if they had simply vanished would bias it, because refinancers are not a random subset (they are disproportionately the borrowers with the most incentive and the best credit). The clean structure is a **discrete-time competing-risks hazard**: for every loan-month, a probability of each outcome, conditional on the loan having survived to the start of that month.

**Terms.** A **hazard** $h^{k}_{i,t}$ is the probability that loan $i$ leaves by cause $k$ in month $t$, given it was alive at the start of $t$. The **person-month** (here, loan-month) **panel** turns each loan's history into one row per month alive: 59 rows of "survived" and a final row with the outcome, for a loan that paid off in month 60. **Lock-in gap** $L_{i,t} = \max(m_{t-1} - c_i,\, 0)$ is how far the market rate sits above the borrower's rate, in percentage points; it is the incentive turned around, measuring how much it would cost to move and take a new mortgage.

### Process, part 1: multinomial logit (the interpretable benchmark)

Each cause $k \in \{S, R, D\}$ gets a linear score (log-odds against surviving),

$$\eta^{k}_{i,t} = \mathbf{z}_{i,t}'\,\boldsymbol\beta^{k},$$

and the scores are turned into probabilities that are positive and sum to less than one by the **softmax**:

$$h^{k}_{i,t} = \frac{e^{\eta^{k}_{i,t}}}{1 + \sum_{k' \in \{S,R,D\}} e^{\eta^{k'}_{i,t}}}, \qquad h^{0}_{i,t} = \frac{1}{1 + \sum_{k'} e^{\eta^{k'}_{i,t}}}.$$

The ratio $h^{k}/h^{0} = e^{\eta^{k}}$ is the odds of cause $k$ versus surviving, so each coefficient is a log odds ratio: a coefficient of $-0.15$ multiplies those odds by $e^{-0.15} = 0.86$.

**Sale equation** (turnover):

$$\eta^{S}_{i,t} = \beta^S_0 + \underbrace{\beta^S_{L1} \min(L_{i,t}, 3) + \beta^S_{L2} \max(L_{i,t} - 3, 0)}_{\text{lock-in, piecewise linear}} + \underbrace{\beta^S_{\text{age}} \max\!\Big(0, \tfrac{36 - \text{age}_{i,t}}{36}\Big)}_{\text{seasoning ramp}} + \underbrace{\beta^S_{\text{LTV}} \max(\text{CLTV}_{i,t} - 0.8, 0)}_{\text{equity constraint}} + \gamma_{\text{moy}(t)} + \beta_f\, f_{j(i),t}.$$

- The **lock-in** term is a **linear spline** (straight-line pieces joined at a **knot**, here at 3 percentage points): each point of rate gap cuts mobility, but less so beyond 3 points, because the borrowers still moving at that point are those who have to (job change, divorce, death).
- The **seasoning ramp** says new borrowers rarely move in their first three years.
- **Equity constraint**: a borrower with little equity cannot easily fund a down payment on the next house.
- $\gamma_{\text{moy}}$ is a **seasonality** effect for each month of the year.
- $\beta_f f_{j(i),t}$ is the local housing market from 5.3.

**Refinance equation:**

$$\eta^{R}_{i,t} = \beta^R_0 + \beta^R_{G1}\,\operatorname{clip}(G_{i,t} + 0.5,\ 0,\ 2) + \beta^R_{G2} \max(G_{i,t} - 1.5,\ 0) + \beta^R_{b}\log\frac{b_{i,t}}{300{,}000} + \beta^R_{\text{LTV}}\max(\text{CLTV}_{i,t} - 0.8, 0) + \beta^R_{\text{FICO}}(\text{FICO}_i - 740) + \beta^R_{\text{TX}}\,\text{TX}_i.$$

$\operatorname{clip}(x, 0, 2)$ means $x$ cut off below at 0 and above at 2. Together with the last incentive term this traces the familiar **S-curve**: flat at a cash-out floor while the loan is out of the money, steep from $G = -0.5$ to $+1.5$, then flattening. Larger loans refinance more readily (fixed closing costs are a smaller share of the savings); Texas loans less (Texas law restricts cash-out refinancing to 80% LTV).

**Default equation:** $\eta^{D}_{i,t} = \beta^D_0 + \beta^D_{\text{FICO}}(\text{FICO}_i - 740) + \beta^D_{\text{LTV}}\max(\text{CLTV}_{i,t} - 0.8, 0)$.

**Estimation.** Maximum likelihood on the loan-month panel with the soft labels from 5.1:

$$\ell(\boldsymbol\beta) = \sum_{i,t}\ \sum_{k \in \{0,S,R,D\}} y^{k}_{i,t}\,\log h^{k}_{i,t}(\boldsymbol\beta),$$

where for an ambiguous payoff month $y^S = \omega_e$ and $y^R = 1 - \omega_e$, and for every other month exactly one $y^k$ is 1. This is an ordinary multinomial logit, fit by Newton's method; fractional outcomes change nothing in the mechanics. The panel is very large (tens of millions of loan-months per vintage), so in practice one keeps all payoff months and a random sample of survival months, adding the log of the sampling rate as an offset to the survival category.

### Process, part 2: gradient boosting with monotone constraints (the production model)

The logit forces each effect to be additive on the log-odds scale. Real turnover has interactions: lock-in bites harder for younger borrowers with small balances than for empty-nesters with large equity; a weak local market matters more for recently purchased homes. A **gradient-boosted** multiclass model (an ensemble of shallow decision trees, each fit to what the previous trees got wrong) captures these, starting from the logit's scores as its baseline so it only learns corrections.

Unconstrained trees will happily learn noise that reverses economics in thin corners of the data, for example a sale hazard that rises with the lock-in gap between 4 and 4.5 points. That would be fatal in 5.5, which evaluates the model at rate levels far from today's. So we impose **monotone constraints**: the sale score must be non-increasing in $L$, the refinance score non-decreasing in $G$, the default score non-decreasing in CLTV. Concretely, when a tree splits on a constrained variable, the leaf values must respect the order. Suppose a tree's first split on $L$ produces three leaves with log-odds corrections $+0.04$ (for $L < 2$), $-0.05$ (for $2 \le L < 3.5$), $-0.02$ (for $L \ge 3.5$). The last pair violates "non-increasing," so the constrained fit pools them to their weighted average (say $-0.035$ each), giving $+0.04, -0.035, -0.035$. The economics survive; the noise does not.

In this walkthrough, the logit's numbers are carried forward so that every figure can be checked by hand. In production the boosted model replaces them, and the logit stays as the benchmark and as the explanation layer for the report.

### Worked example: Loan L in November 2026

Suppose the fitted coefficients (monthly log-odds) are:

| Coefficient | Value | Coefficient | Value |
|---|---|---|---|
| $\beta^S_0$ | −5.20 | $\beta^R_0$ | −8.00 |
| $\beta^S_{L1}$, $\beta^S_{L2}$ | −0.15, −0.06 | $\beta^R_{G1}$, $\beta^R_{G2}$ | 2.4, 0.6 |
| $\beta^S_{\text{age}}$ | −1.2 | $\beta^R_{b}$ | 0.35 |
| $\beta^S_{\text{LTV}}$ | −2.0 | $\beta^R_{\text{LTV}}$ | −2.5 |
| $\beta_f$ | 0.15 | $\beta^R_{\text{FICO}}$, $\beta^R_{\text{TX}}$ | 0.004, −0.25 |
| $\gamma_{\text{Nov}}$ | −0.163 (November is 15% below an average month) | $\beta^D_0$, $\beta^D_{\text{FICO}}$, $\beta^D_{\text{LTV}}$ | −8.60, −0.012, 3.0 |

The sale intercept means an unlocked, seasoned, well-equitied loan in an average month and average market moves with probability about $e^{-5.20} \approx 0.55\%$ a month, about 6.4% a year: roughly the pre-2022 national turnover rate.

**Loan L's features.** Market rate $m_{\text{Oct}} = 7.30\%$, so $G = 2.75 - 7.30 = -4.55$ and $L = 4.55$. Age 61 months. Balance \$283,186.87 against an AVM value of \$414,000, so $\text{CLTV} = 0.684$. FICO 765. Texas. Turnover index for November from 5.3: $f_{\text{TX},\text{Nov}} = -0.563$.

**Sale score:**

| Term | Calculation | Value |
|---|---|---|
| Intercept | | −5.200 |
| Lock-in | $-0.15 \times 3 - 0.06 \times 1.55$ | −0.543 |
| Seasoning | age ≥ 36 | 0 |
| Equity | CLTV < 0.8 | 0 |
| November | | −0.163 |
| Turnover index | $0.15 \times (-0.563)$ | −0.084 |
| $\eta^S$ | | **−5.990** |

**Refinance score:** $-8.00 + 0$ (incentive below −0.5, on the floor) $+ 0.35\log(283{,}187/300{,}000) = -0.020$, $+ 0.004 \times 25 = +0.100$, $-0.25$ (Texas): $\eta^R = -8.170$.

**Default score:** $-8.60 - 0.012 \times 25 = -8.900$.

**Softmax.** $e^{\eta^S} = 0.002504$, $e^{\eta^R} = 0.000283$, $e^{\eta^D} = 0.000136$; the denominator is $1.002923$. So

$$h^S_{L,\text{Nov}} = 0.250\%, \qquad h^R_{L,\text{Nov}} = 0.028\%, \qquad h^D_{L,\text{Nov}} = 0.014\%,$$

a total payoff probability of 0.291% in November. Sale is 86% of it.

**What lock-in costs.** The lock-in term alone multiplies the sale odds by $e^{-0.543} = 0.58$. If Loan L had no rate gap (and the local market were average), its November sale hazard would be 0.466% instead of 0.250%.

**How Loan L responds to rates.** Holding everything else fixed (turnover index at 0), as the market rate falls:

| Market rate $m$ | 7.3% | 6.0% | 5.0% | 4.0% | 3.0% | 2.5% | 2.0% |
|---|---|---|---|---|---|---|---|
| Sale hazard (%/mo) | 0.272 | 0.294 | 0.333 | 0.387 | 0.449 | 0.466 | 0.464 |
| Refi hazard (%/mo) | 0.028 | 0.028 | 0.028 | 0.028 | 0.051 | 0.170 | 0.562 |

Two different mechanisms. A rally first **loosens lock-in**, which raises the sale hazard gradually all the way down; refinancing does not wake up until the market is within half a point of the loan's 2.75%. For a 2021 loan, the rate path that turns it into a refinance candidate is one almost no scenario reaches, which is why the turnover model is the whole valuation story for discount pools.

### Output

A function: given a loan's static features and a future month's mortgage rate, age, CLTV, calendar month and turnover index, return $(h^S, h^R, h^D)$. Also the fitted coefficients and (for the boosted model) feature attributions, for the report.

### Hand-off

To 5.4 (aggregation) and, inside it, to 5.5, which calls the function on every loan, every month, every rate path. The same hazard model is a **shared module**: 4.3 uses its survival curve $S(t) = \prod_{u \le t} h^{0}_{u}$ to value assumable loans, and summing the sale hazard by 3-digit zip gives brokerages a map of where sellers are most likely to appear.

### Validation

Out-of-time: fit on 2015 to 2023, predict 2024 to 2026 monthly payoffs by cause. Calibration by decile of predicted sale hazard (predicted vs. sale-labeled payoffs). Because 2022 to 2026 is the first long lock-in episode in the data, check specifically that predicted sale hazards at lock-in gaps above 3 points match what happened, since the earlier data barely visit that region. And compare the boosted model to the logit: if boosting does not beat the logit's out-of-time likelihood by a clear margin, ship the logit.

---

## Step 5.3: Local nowcasting (dynamic factor model + Kalman filter)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Pending home sales, year-over-year log change, standardized | $x_{1,t}$ | Redfin / Realtor.com aggregates, monthly |
| New listings, same transformation | $x_{2,t}$ | Same |
| Net domestic migration | $x_{3,t}$ | IRS migration data, USPS change-of-address aggregates (published with a lag) |
| Job flows (hires plus quits rate) | $x_{4,t}$ | BLS JOLTS state estimates, QCEW |
| Region | $j$ | One filter per state or MSA |

Each indicator is **standardized**: transformed to year-over-year log change and scaled to mean 0 and variance 1 over the training history. So $x = -1.1$ means "1.1 standard deviations below normal for this time of year."

### Why a factor model

Turnover data from the loans themselves arrive late and are noisy: a single pool's monthly payoffs are a handful of events, and the loan-level files are released weeks after month end. Meanwhile the housing market produces faster, cleaner signals: pending sales (contracts signed, which close one or two months later), new listings (sellers deciding to move), migration and job changes (reasons to move). No single indicator *is* turnover, but they all move together with it. A **dynamic factor model** says each indicator is a noisy reading of one unobserved common force $f_t$, the region's turnover pressure, and the **Kalman filter** estimates that force in real time, even when some indicators are missing.

### Process

**Measurement equations** (one per indicator $\ell = 1, \dots, 4$):

$$x_{\ell,t} = \lambda_\ell\, f_t + e_{\ell,t}, \qquad e_{\ell,t} \sim \mathcal{N}(0, \psi_\ell),$$

where $\lambda_\ell$ is the **loading** (how strongly indicator $\ell$ reflects the common factor) and $\psi_\ell$ its idiosyncratic noise variance.

**State equation:**

$$f_t = \phi\, f_{t-1} + u_t, \qquad u_t \sim \mathcal{N}(0, q).$$

$\phi$ is the month-to-month persistence. The factor has no natural scale, so we fix its long-run variance at 1, which pins $q = 1 - \phi^2$. With unit-variance indicators, $\psi_\ell = 1 - \lambda_\ell^2$.

**The Kalman filter** carries a current estimate $f_{t|t}$ ("the factor at $t$ given data through $t$") and its variance $\Sigma_{t|t}$ forward in two steps each month:

- **Predict:**
$$f_{t|t-1} = \phi\, f_{t-1|t-1}, \qquad \Sigma_{t|t-1} = \phi^2\, \Sigma_{t-1|t-1} + q.$$
- **Update** with whichever indicators arrived. Because there is one factor and the noises are independent, the update has a transparent **precision-weighted** form (precision = 1 / variance):
$$\frac{1}{\Sigma_{t|t}} = \frac{1}{\Sigma_{t|t-1}} + \sum_{\ell \in \text{obs}(t)} \frac{\lambda_\ell^2}{\psi_\ell}, \qquad f_{t|t} = \Sigma_{t|t}\left( \frac{f_{t|t-1}}{\Sigma_{t|t-1}} + \sum_{\ell \in \text{obs}(t)} \frac{\lambda_\ell\, x_{\ell,t}}{\psi_\ell} \right).$$

Read the update as a weighted average of the prediction and each indicator's implied value of $f$ (which is $x_\ell / \lambda_\ell$), each weighted by how informative it is. **Missing data handle themselves**: an indicator that has not been published simply drops out of $\text{obs}(t)$. That is how the filter copes with **staggered releases** (pending sales out mid-month, migration months later) and a **ragged edge** at the end of the sample. When a late indicator does arrive, rerun the update for that month.

This is algebraically identical to the textbook Kalman gain form $f_{t|t} = f_{t|t-1} + K_t\,(\mathbf{x}_t - \boldsymbol\lambda f_{t|t-1})$ with $K_t = \Sigma_{t|t-1}\boldsymbol\lambda'(\boldsymbol\lambda \Sigma_{t|t-1} \boldsymbol\lambda' + \Psi)^{-1}$, restricted to the observed rows.

**Forecasting** $h$ months ahead is the predict step repeated with no update:

$$f_{t+h|t} = \phi^h f_{t|t}, \qquad \Sigma_{t+h|t} = \phi^{2h}\Sigma_{t|t} + (1 - \phi^{2h}),$$

so the forecast decays to 0 (the long-run average) and its variance rises toward 1 (no information).

**Estimation.** $\phi$ and the $\lambda_\ell$ are fit by maximum likelihood using the filter's own one-step prediction errors (the **prediction-error decomposition**): with $\mathbf{v}_t = \mathbf{x}_t - \boldsymbol\lambda f_{t|t-1}$ and $F_t = \boldsymbol\lambda\Sigma_{t|t-1}\boldsymbol\lambda' + \Psi$ over observed rows,

$$\log \mathcal{L} = -\tfrac{1}{2}\sum_t \big[ \log\det F_t + \mathbf{v}_t' F_t^{-1} \mathbf{v}_t \big] + \text{const},$$

maximized numerically. The sign of $f$ is fixed by requiring $\lambda_1 > 0$ (more pending sales means more turnover).

**Linking to 5.2, and a trap to avoid.** 5.2 is trained with $f$ as a feature. Use the **filtered** values $f_{t|t}$ that would have been available in real time, not the **smoothed** values that use later data. Training on smoothed values gives the hazard model a sharper signal than it will ever see when forecasting, and its coefficient $\beta_f$ will be too large.

### Worked example: Texas, September 2026

Fitted parameters: $\phi = 0.85$, so $q = 1 - 0.85^2 = 0.2775$, and

| Indicator | $\lambda_\ell$ | $\psi_\ell = 1 - \lambda_\ell^2$ | Precision contribution $\lambda_\ell^2/\psi_\ell$ |
|---|---|---|---|
| Pending sales | 0.90 | 0.19 | 4.263 |
| New listings | 0.70 | 0.51 | 0.961 |
| Migration | 0.50 | 0.75 | 0.333 |
| Job flows | 0.60 | 0.64 | 0.563 |

Pending sales dominate because they are the closest thing to turnover itself.

**Start:** August's filtered estimate $f_{\text{Aug}|\text{Aug}} = -0.30$ with $\Sigma = 0.12$.

**Predict September:** $f_{\text{Sep}|\text{Aug}} = 0.85 \times (-0.30) = -0.255$; $\Sigma_{\text{Sep}|\text{Aug}} = 0.7225 \times 0.12 + 0.2775 = 0.364$ (prior precision $1/0.364 = 2.746$).

**September data:** pending sales $-1.10$, new listings $-0.20$, job flows $-0.50$; migration **not yet published**.

**Update:**

$$\frac{1}{\Sigma_{\text{Sep}|\text{Sep}}} = 2.746 + 4.263 + 0.961 + 0.563 = 8.532, \qquad \Sigma_{\text{Sep}|\text{Sep}} = 0.117,$$

$$f_{\text{Sep}|\text{Sep}} = 0.117 \times \Big( \underbrace{\tfrac{-0.255}{0.364}}_{-0.700} + \underbrace{\tfrac{0.90 \times (-1.10)}{0.19}}_{-5.211} + \underbrace{\tfrac{0.70 \times (-0.20)}{0.51}}_{-0.275} + \underbrace{\tfrac{0.60 \times (-0.50)}{0.64}}_{-0.469} \Big) = 0.117 \times (-6.654) = -0.780.$$

A sharp drop, from a predicted −0.26 to −0.78 (standard deviation 0.34), driven almost entirely by the pending-sales collapse. Listings barely fell, which is itself informative: sellers are still listing, buyers are not signing.

**When migration arrives** a month later at $-0.80$, rerunning September's update adds $0.333$ to the precision and $0.5 \times (-0.80)/0.75 = -0.533$ to the sum, revising the estimate to $-0.811$ (variance 0.113). Small, because migration is the least informative indicator.

**Forecast path** (from $f_{\text{Sep}|\text{Sep}} = -0.780$):

| Month | Oct | Nov | Dec | Feb | Apr | Sep 2027 |
|---|---|---|---|---|---|---|
| $h$ | 1 | 2 | 3 | 5 | 7 | 12 |
| $f_{t+h}$ | −0.663 | −0.563 | −0.479 | −0.346 | −0.250 | −0.111 |
| sd | 0.60 | 0.73 | 0.82 | 0.91 | 0.95 | 0.99 |

November's $-0.563$ is the value used for Loan L in 5.2. Its effect there is a multiplier $e^{0.15 \times (-0.563)} = 0.92$ on the sale odds: an 8% cut in November turnover, fading to about 2% a year out.

### Output

For each region, the filtered turnover index $f_{t|t}$, its variance, and its forecast path. Also, per indicator, the **news**: how much each release moved the estimate (here, pending sales contributed −5.21 of the −6.65 total), which the report uses to explain turnover surprises.

### Hand-off

To 5.2 as the feature $f_{j(i),t}$, in history (filtered values for training) and in forecasts (the decaying path, which 5.5 uses on every rate path).

### Validation

Two checks. Does the index predict next-month sale-labeled payoffs out of sample, beyond what 5.2 already knows without it (the improvement in out-of-time log-likelihood from adding $\beta_f f$)? And is the filter calibrated: about 80% of the following month's standardized pending-sales readings should fall inside the predicted 80% band $\lambda_1 f_{t+1|t} \pm 1.28\sqrt{\lambda_1^2\Sigma_{t+1|t} + \psi_1}$.

---

## Step 5.4: Aggregation and calibration

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Loan hazards for every loan and future month | $h^{S}_{i,t}, h^{R}_{i,t}, h^{D}_{i,t}$ | 5.2 (with $f$ from 5.3) |
| Loan balances and amortization schedules | $b_{i,t}$ | Agency loan-level data |
| Curtailment rate | $\kappa$ | Historical average by loan age |
| Pool's own recent payoff history | $A_p$, $n_p$ | Agency monthly pool factors and loan-level files |
| Between-pool spread of true speed differences | $v$ | Estimated across all pools |

### Process, part 1: loans to pools

An investor receives principal in dollars, so the pool's prepayment rate is the **balance-weighted** average of its loans' hazards. Let $\tilde b_{i,t}$ be loan $i$'s balance after this month's scheduled principal (the balance that is *available* to prepay). Then

$$\text{SMM}_{p,t} = \frac{\sum_{i \in p} \tilde b_{i,t}\,\big(h^{S}_{i,t} + h^{R}_{i,t} + h^{D}_{i,t}\big) + \kappa \sum_{i \in p} \tilde b_{i,t}\,\big(1 - h^{S}_{i,t} - h^{R}_{i,t} - h^{D}_{i,t}\big)}{\sum_{i \in p} \tilde b_{i,t}},$$

where the second term is curtailments on loans that did not pay off ($\kappa = 0.04\%$ a month for loans older than two years). Default is included because a default buyout returns principal to the investor at par. Then

$$\text{CPR}_{p,t} = 1 - (1 - \text{SMM}_{p,t})^{12}.$$

Two points that matter in practice. Balances evolve: as loans pay off, the survivors' weights grow, so we track expected balances month by month rather than averaging hazards once. And **heterogeneity matters because the hazard is nonlinear**: the pool's speed is not the hazard of an average loan. A pool with half its loans at 2.25% and half at 3.25% is not a pool of 2.75% loans. So we aggregate loan by loan (or, for speed, over a few dozen buckets of similar loans), never from pool averages.

### Process, part 2: credibility weighting

Every pool has something the model does not see: a servicer that solicits aggressively, a neighborhood with a big employer moving in, borrowers who are older than average. The pool's own track record reveals some of it, through the **actual-to-expected ratio**

$$R_p = \frac{A_p}{E_p} = \frac{\text{actual payoffs over the last 12 months}}{\text{payoffs the model predicted for those months}}.$$

But a single pool's 12 months contain only a couple of dozen payoffs, so $R_p$ is mostly noise. **Credibility theory** (from actuarial science) answers how much to believe it. The Bühlmann credibility weight is

$$Z_p = \frac{n_p}{n_p + K}, \qquad \mu_p = 1 + Z_p\,(R_p - 1),$$

where $n_p$ is the pool's exposure (loan-months observed) and $\mu_p$ is the pool-specific speed multiplier we apply to future hazards. $K$ is the ratio of **within-pool noise** to **between-pool signal**:

$$K = \frac{\text{expected process variance per loan-month}}{\text{variance of true pool multipliers across pools}}.$$

For a payoff indicator with average monthly probability $\bar h$, the process variance is $\bar h (1-\bar h) \approx \bar h$; if true pool multipliers vary across pools with variance $v$, the variance of their true payoff rates is $\bar h^2 v$. So $K \approx 1/(\bar h\, v)$. $v$ is estimated across many pools: the observed variance of $R_p$ minus the part explained by Poisson noise.

This is the same shrinkage idea as partial pooling in the Escrow service's tax-rate model: a little data pulls a little, a lot of data pulls a lot.

### Worked example: Loan L to Pool T

**Loan L in November.** Hazards from 5.2 total 0.2914%. Adding curtailment on the surviving balance,

$$\text{SMM}_{L} = 0.2914\% + (1 - 0.2914\%) \times 0.04\% = 0.331\%, \qquad \text{CPR}_{L} = 1 - (1 - 0.00331)^{12} = 3.90\%.$$

(In dollars: November's scheduled payment of \$1,306.37 contains \$648.97 interest and \$657.40 scheduled principal, leaving \$282,529.47; the expected prepayment is $0.331\% \times 282{,}529 = \$936$.)

**Credibility for Pool T.** Over October 2025 to September 2026, Pool T had $n_T = 7{,}350$ loan-months and 27 full payoffs; the model, including the turnover index, predicted 22.6. So

$$R_T = \frac{27}{22.6} = 1.195, \qquad \bar h = \frac{22.6}{7{,}350} = 0.00307.$$

Across all 2021 discount pools, true speed multipliers vary with a standard deviation of about 15%, so $v = 0.15^2 = 0.0225$ and

$$K = \frac{1}{0.00307 \times 0.0225} = 14{,}454 \text{ loan-months}, \qquad Z_T = \frac{7{,}350}{7{,}350 + 14{,}454} = 0.337,$$

$$\mu_T = 1 + 0.337 \times 0.195 = 1.066.$$

Pool T ran 19.5% faster than the model last year; we believe 6.6% of that is real. The pool would need about 14,000 loan-months (two years at its current size) before we gave its own history half the weight. The **uncertainty** that remains about the true multiplier is $\sqrt{(1 - Z_T)\,v} = \sqrt{0.663 \times 0.0225} = 0.12$: we think Pool T's turnover is 6.6% above model, give or take 12%. That number comes back in 5.5.

**Pool T, month 1 (November 2026).** Running all 600 loans through 5.2 (each with its own rate, balance, FICO and value), weighting by balance, and applying $\mu_T$ to the sale and refinance hazards gives, per \$100 of current balance:

| Component | \$ per \$100 |
|---|---|
| Scheduled principal | 0.229 |
| Sale payoffs | 0.265 |
| Refinance payoffs | 0.030 |
| Default buyouts | 0.017 |
| Curtailments | 0.040 |
| **Prepaid total** | **0.352** |

$$\text{SMM}_{T,\text{Nov}} = \frac{0.352}{100 - 0.229} = 0.353\%, \qquad \text{CPR}_{T,\text{Nov}} = 4.16\%.$$

The pool runs faster than Loan L (3.90%) because some of its loans have higher note rates (less lock-in) and lower LTVs, and because of $\mu_T$.

**The projected CPR curve** (along today's forward rate curve):

| Month | 1 (Nov 26) | 6 (Apr 27) | 12 (Oct 27) | 24 | 60 | 120 | 240 |
|---|---|---|---|---|---|---|---|
| Pool T CPR (%) | 4.16 | 5.05 | 4.75 | 4.76 | 4.70 | 4.64 | 4.53 |
| TBA 2.0 cohort CPR (%) | 4.23 | 4.99 | 4.63 | 4.62 | 4.55 | 4.48 | 4.35 |

The 12-month CPR (the share of today's balance, net of scheduled principal, prepaid over the next year) is **4.86%** for Pool T vs. 4.80% for the generic cohort. Decomposed:

| Model version | 12-month CPR |
|---|---|
| National model, no turnover index, no credibility | 4.74% |
| + Texas turnover index (5.3) | 4.61% (−0.13) |
| + Pool T credibility multiplier | 4.86% (+0.25) |

The weak Texas market and Pool T's own fast history pull in opposite directions; the pool's history wins, modestly. Note the shape of the curve: a seasonal swing (spring peak, winter trough) on a slowly declining trend. The decline comes from the forward curve, which implies mortgage rates drifting up from 7.3% to about 7.8% over ten years and deepening lock-in.

### Output

For each pool: $\text{SMM}_{p,t}$ and $\text{CPR}_{p,t}$ for every future month, **as a function of the rate path**, split by cause; and the credibility multiplier $\mu_p$ with its uncertainty. Cohort curves come from the same sum over all pools in a coupon-and-vintage cohort.

### Hand-off

To 5.5, which evaluates the pool's prepayment function along each simulated rate path. The single forward-curve projection above goes straight to the report as the "base case" CPR curve.

### Validation

Pool-level out-of-time backtests: from data through month $t$, project each pool's next-12-month CPR and compare with what happened. Check that credibility helps: the forecast error should be lower with the fitted $K$ than with $Z = 0$ (ignore pool history) or $Z = 1$ (trust it fully). If not, re-estimate $v$.

---

## Step 5.5: Valuation (Hull-White rate paths + cash-flow engine + OAS)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Treasury zero-coupon curve today | $P(0,T)$, $F(0,t)$ | Fed GSW zero curve (or the SOFR swap curve) |
| Rate volatility | $a$, $\sigma$ | Calibrated to swaption implied volatilities |
| Mortgage–Treasury spread | $s^{M}$ | PMMS minus 10-year Treasury, today |
| Pool prepayment function | $\text{SMM}_{p,t}(\text{rate path})$ | 5.2 → 5.4 |
| Pool coupon, loan schedules | $c^{\text{pt}}$ | Pool disclosures |
| Market prices: TBA and specified pool | $\text{Px}^{\text{mkt}}$ | Dealer runs, trading platforms |

**Terms.** A **zero-coupon price** $P(0,T)$ is today's price of \$1 paid at time $T$; the **instantaneous forward rate** $F(0,t)$ is the rate the curve implies for borrowing over an instant at time $t$. A **short rate** $r_t$ is the interest rate over the next instant. A **swaption** is an option to enter an interest-rate swap; its price reveals how volatile the market expects rates to be. The **option-adjusted spread (OAS)** is the constant spread over the simulated short rates at which the average discounted cash flow equals the market price. It is "option-adjusted" because the cash flows already reflect how prepayments respond to rates on each path.

### Why simulate rate paths

A pool's cash flows depend on the future path of mortgage rates, and that dependence is **nonlinear** and **path-dependent**: a rally loosens lock-in and wakes up refinancing; a selloff deepens lock-in; and what happens in year five depends on how many loans survived years one to four. So the value is not the discounted cash flow along one expected path. It is the average over many paths of each path's own discounted cash flow. That requires a rate model that can generate realistic paths consistent with today's prices.

### Process, part 1: the Hull-White model

The **Hull-White one-factor model** describes the short rate as

$$dr_t = \big(\theta(t) - a\, r_t\big)\,dt + \sigma\, dW_t.$$

- $a$ is the **mean-reversion speed**: how fast the rate is pulled back toward its target. $a = 0.05$ means a half-life of $\ln 2 / 0.05 \approx 14$ years: shocks are long-lived.
- $\sigma$ is the **volatility** of the short rate, in rate units per $\sqrt{\text{year}}$. $\sigma = 0.011$ means about 110 basis points of standard deviation over a year.
- $dW_t$ is the increment of a **Brownian motion** (a random walk in continuous time; over a step of length $\Delta$ it is $\sqrt{\Delta}$ times a standard normal draw).
- $\theta(t)$ is a time-varying target chosen so the model **reproduces today's yield curve exactly**:
$$\theta(t) = \frac{\partial F(0,t)}{\partial t} + a\,F(0,t) + \frac{\sigma^2}{2a}\big(1 - e^{-2at}\big).$$

This is the same rate simulator listed in the shared-components table for 4.3, and a richer cousin of the AR(1) rate paths in the Labor Cost service (1.5): the difference is that Hull-White is **arbitrage-free**, meaning that if you price any Treasury bond by averaging its discounted cash flows over the paths, you get its market price. That property is essential when the output is a spread to market prices.

**Simulation.** Write $r_t = \alpha(t) + X_t$ with $\alpha(t) = F(0,t) + \frac{\sigma^2}{2a^2}(1 - e^{-at})^2$. Then $X$ is a simple mean-reverting process starting at 0 that can be stepped exactly, month by month ($\Delta = 1/12$):

$$X_{t+\Delta} = e^{-a\Delta} X_t + \sigma\sqrt{\frac{1 - e^{-2a\Delta}}{2a}}\;\varepsilon_{t+\Delta}, \qquad \varepsilon \sim \mathcal{N}(0,1).$$

We use $N = 2{,}000$ paths of 300 months, in **antithetic** pairs (each path's normal draws are also used with flipped signs), which cancels much of the simulation noise.

**From short rate to mortgage rate.** Prepayments respond to the mortgage rate, which tracks the 10-year Treasury yield, not the short rate. Hull-White gives zero-coupon bond prices on each path in closed form:

$$P(t,T) = A(t,T)\,e^{-B(t,T)\,r_t}, \qquad B(t,T) = \frac{1 - e^{-a(T-t)}}{a},$$
$$\ln A(t,T) = \ln\frac{P(0,T)}{P(0,t)} + B(t,T)\,F(0,t) - \frac{\sigma^2}{4a}\big(1 - e^{-2at}\big)B(t,T)^2.$$

The 10-year par yield (semiannual coupons) on the path at month $t$ is then

$$y^{(10)}_t = \frac{2\,\big[1 - P(t, t+10)\big]}{\sum_{k=1}^{20} P(t, t + k/2)},$$

and the mortgage rate is $m_t = y^{(10)}_t + s^{M}$, holding today's spread fixed. (A refinement lets $s^{M}$ mean-revert toward its historical average; today's spread is unusually wide.)

### Process, part 2: cash flows on each path

On path $n$, month by month, for every loan in the pool: compute the scheduled principal from its note rate and remaining term; evaluate the hazards from 5.2 using that path's lagged mortgage rate $m_{n,t-1}$, the loan's current age and CLTV, the calendar month, and the decaying turnover index $f_{t|T} = \phi^{h} f_{T|T}$; multiply sale and refinance hazards by $\mu_p$; prepay that share of the balance; carry the surviving balance forward. The investor's cash flow per \$100 of today's balance is

$$\text{CF}_{n,t} = \underbrace{\frac{c^{\text{pt}}}{12}\,B_{n,t-1}}_{\text{interest at the pass-through coupon}} + \underbrace{\text{scheduled}_{n,t} + \text{prepaid}_{n,t}}_{\text{principal}},$$

where $B_{n,t-1}$ is the pool balance at the start of month $t$. The spread between WAC and coupon (here 0.86%) goes to the servicer and guarantor and never reaches the investor.

### Process, part 3: price, OAS, and what to compare

**Price at spread $s$:**

$$\text{Px}(s) = \frac{1}{N}\sum_{n=1}^{N}\sum_{t=1}^{300} \text{CF}_{n,t}\; \exp\!\Big(-\sum_{u < t} r_{n,u}\,\Delta - r_{n,t}\,\delta\Big)\; e^{-s\,\tau_t},$$

where $\delta = 0.83/12$ years is the extra wait from month end to the 25th of the next month, and $\tau_t = (t + 0.83)/12$ is the payment time in years. The **OAS** is the $s$ that solves $\text{Px}(s) = \text{Px}^{\text{mkt}}$, found by bisection since $\text{Px}(s)$ falls steadily as $s$ rises. (Accrued interest and exact settlement dates are ignored here.)

Three comparisons turn the OAS into a decision:

1. **OAS vs. the TBA cohort.** Compute the generic cohort's OAS at the TBA price with the same engine and the *same random paths* (**common random numbers**, so simulation noise cancels in the difference). A specified pool whose OAS exceeds its cohort's is **cheap**: at its price, it pays more spread for the same rate risk.
2. **Fair pay-up.** Price the pool at the cohort's OAS: $\text{Px}_p(\text{OAS}_{\text{cohort}}) - \text{Px}^{\text{TBA}}$ is the pay-up our speeds justify. Compare to the pay-up the market charges.
3. **Implied speed.** Scale the pool's hazards by a factor $k$ until its OAS equals the cohort's. That $k^*$ is the speed the market is pricing in, which can be compared directly with our projection and with dealer prepayment projections.

And two risk numbers: the **ZV spread** (zero-volatility spread: the spread over the forward curve that prices the forward-path cash flows) minus the OAS is the **option cost**, how much value the borrowers' prepayment options take away; and **effective duration and convexity**, from repricing at fixed OAS with the whole curve shifted by $\pm\Delta y$:

$$D^{\text{eff}} = \frac{\text{Px}_{-\Delta y} - \text{Px}_{+\Delta y}}{2\,\text{Px}_0\,\Delta y}, \qquad C^{\text{eff}} = \frac{\text{Px}_{-\Delta y} + \text{Px}_{+\Delta y} - 2\,\text{Px}_0}{\text{Px}_0\,\Delta y^2}.$$

### Worked example: Pool T

**Curve and calibration.** Today's Treasury zero curve (continuously compounded): 3.85% at 3 months, 4.17% at 2 years, 4.53% at 5 years, 4.84% at 10 years, 5.14% at 30 years; 10-year par yield 4.85%. With PMMS at 7.30%, $s^{M} = 7.30 - 4.85 = 2.45$ percentage points. Hull-White: $a = 0.05$, $\sigma = 0.011$. As a check on the paths, the simulated average discount factor to 10 years is 0.6168 against the curve's 0.6162, inside simulation error. Across paths, the mortgage rate five years out has P10 5.6%, median 8.0%, P90 10.4%: a wide fan, because rate volatility compounds over five years and mean reversion is slow.

**One month of cash flow.** On every path, November's cash flow is the same (rates have not moved yet, and the incentive uses October's rate): interest $2.0\%/12 \times 100 = 0.167$ plus principal 0.581 (scheduled 0.229 + prepaid 0.352, as in 5.4), total **0.748** per \$100, paid December 25 and discounted by 0.9942.

**Three paths, side by side** (chosen at the 5th, 50th and 95th percentile of the mortgage rate in five years):

| Path | Mortgage rate in 5 yrs | Lowest mortgage rate on path | CPR in year 5 | WAL (yrs) | PV at the solved OAS |
|---|---|---|---|---|---|
| Rally | 4.99% | 3.10% | 5.42% | 8.95 | 96.15 |
| Middle | 7.96% | 6.76% | 4.72% | 9.56 | 74.29 |
| Selloff | 11.05% | 6.91% | 4.53% | 9.80 | 65.71 |

**WAL** (weighted-average life) is the average time until each dollar of principal is returned. Notice how little speeds move: even in the rally path, year-5 CPR rises only from 4.7% to 5.4%. A 2.86% pool is so far out of the money that the rally mostly works through **discounting** (lower rates raise the present value) and only slightly through faster principal. That is why discount pools are prized for **convexity**: their prepayment option barely bites.

**OAS.** With TBA UMBS 2.0 at 77-28, the cohort's OAS is **34.9 bp**. Pool T at 78-00 has an OAS of **36.2 bp**. ZV spreads are 40.7 bp (Pool T) and 39.2 bp (cohort), so the option cost is about 4.5 bp for both. Simulation error on the price is about 0.03 points (1/32), and smaller on the difference thanks to common random numbers.

**Fair pay-up.** At the cohort's 34.9 bp OAS, Pool T is worth 78.072, which is $78.072 - 77.875 = 0.197$, or about **6.3/32**, over TBA. The market charges 4/32. Pool T is **cheap by about 2.3/32**, which is the same statement as its 1.3 bp of extra OAS.

**What is the market pricing in?** Rerunning Pool T with its hazards scaled by $k$:

| Speed multiplier $k$ | 0.85 | 1.00 | 1.15 | 1.30 |
|---|---|---|---|---|
| 12-month CPR | 4.03% | 4.61% | 5.19% | 5.76% |
| OAS at 78-00 (bp) | 15.5 | 29.8 | 44.4 | 59.2 |

Each 1 CPR of extra speed adds about 25 bp of OAS: $(59.2 - 15.5)/(5.76 - 4.03) = 25.3$. With an effective duration of 7.4 and a price of 78, 25 bp is worth about $7.4 \times 0.0025 \times 78 \approx 1.4$ points, which is the "1½ points per CPR" quoted at the start. Interpolating, the cohort's 34.9 bp is reached at $k^* = 1.052$, an implied 12-month CPR of **4.81%**. Our projection is **4.86%**. The market prices Pool T as if its Texas history were worth a 5% speed premium; we think 6.6%.

**How sure are we?** Not very, and the report must say so. The credibility multiplier's own uncertainty is $\pm 12\%$ (5.4), which by the table above is about $\pm 12$ bp of OAS. The 1.3 bp edge is a tenth of that. The honest verdict is "fairly priced, leaning cheap": hold it, and buy more only at 3/32 or less.

**Risk numbers.** Shifting the curve by $\pm 25$ bp at fixed OAS: Pool T's price moves from 79.45 to 76.58, giving **effective duration 7.37** and **effective convexity +0.63** (in the usual units of $C^{\text{eff}}/100$). Positive convexity is unusual for MBS, which are normally negatively convex because rallies trigger refinancing; for a pool this deep in the discount the refinancing option is so far out of the money that the bond's ordinary bond-like convexity dominates.

**Rate scenarios** (whole curve and mortgage rate shifted, priced at fixed OAS):

| Shift | 12-month CPR | 5-year-out CPR | WAL (yrs) | Refi share of prepaid principal | Price change (points) |
|---|---|---|---|---|---|
| −200 bp | 5.57% | 5.68% | 8.86 | 8.9% | +12.4 |
| −100 bp | 5.12% | 5.05% | 9.21 | 6.9% | +6.0 |
| 0 | 4.86% | 4.72% | 9.45 | 6.4% | 0 |
| +100 bp | 4.64% | 4.47% | 9.63 | 6.5% | −5.5 |

Even a 200 bp rally lifts refinancing to under a tenth of prepayments. The extra speed comes from loosened lock-in: turnover rising from about 4.9% toward 5.6%.

### Output

Per pool: OAS, ZV spread and option cost; fair pay-up vs. market pay-up; implied speed multiplier and implied CPR vs. our projection; effective duration and convexity; scenario table; CPR distribution by month across paths.

### Hand-off

To the report. The OAS edges, divided by their uncertainty, give the rich/cheap ranking; the scenario results give rate sensitivity; servicers use the same engine with servicing-fee cash flows instead of pass-through cash flows to value mortgage servicing rights.

### Validation

Price the engine's own benchmarks first: Treasury zero prices and swaption prices recovered from the paths (a check on calibration and simulation). Then, over history: did pools the model ranked cheap subsequently outperform pools it ranked rich, on a hedged basis? Did realized speeds over the following year fall closer to our projection than to the market-implied speed? Those two backtests are the real test of the service.

---

## The report

### Per-pool card (Pool T)

- **Speed:** projected 12-month CPR **4.86%** (P10 4.73%, P90 4.99% across rate paths); about 4.7% over five years on the forward curve. November is the seasonal low (4.2%); April the high (5.1%).
- **Why:** turnover dominates. Of projected prepaid principal, about **80% sale**, 6% refinance, 4% default buyouts, 10% curtailments. Refinancing does not matter unless mortgage rates fall below about 3.3%.
- **Local signal:** Texas turnover index at **−0.78** (pending sales 1.1 standard deviations below normal), worth −0.13 CPR over the next year and fading.
- **Pool history:** ran 19.5% faster than model over the last year; credibility-weighted, we apply +6.6% (uncertainty ±12%).
- **Value:** OAS 36.2 bp vs. 34.9 bp for the TBA cohort; fair pay-up about 6/32 vs. 4/32 charged. Market-implied 12-month CPR 4.81% vs. our 4.86%.
- **Risk:** effective duration 7.4, convexity slightly positive; +12.4 points in a 200 bp rally, −5.5 in a 100 bp selloff.
- **Verdict:** fairly priced, leaning cheap. Hold; add at 3/32 pay-up or less.

### Portfolio table (illustrative, four specified pools run through the same engine and paths)

| Pool | Description | TBA price | Pay-up charged | 12-mo CPR: pool / cohort | Sale share of prepaid | OAS vs. cohort | Fair pay-up | Verdict |
|---|---|---|---|---|---|---|---|---|
| T | UMBS 2.0, 2021, 100% Texas | 77-28 | 4/32 | 4.86% / 4.80% | 80% | **+1.3 bp** | +6.3/32 | Fair, leaning cheap |
| B | UMBS 6.0, 2024, low loan balance (avg \$132k) | 99-28 | 14/32 | 6.78% / 7.03% | 66% | −3.8 bp | +6.8/32 | Rich: pay-up too high |
| C | UMBS 2.0, 2021, California, larger loans (avg \$430k) | 77-28 | 6/32 | 4.81% / 4.80% | 77% | −4.9 bp | −2.6/32 | Rich |
| F | UMBS 3.0, 2022, Florida | 83-16 | 2/32 | 4.45% / 5.00% | 77% | −7.4 bp | −12.1/32 | Rich: worth less than TBA |

Cohort OAS levels at these TBA prices: 2.0s about 35 bp, 3.0s 46.6 bp, 6.0s 79.6 bp (higher coupons carry more prepayment risk and trade wider).

Ranking, cheapest first: **T, B, C, F.** Each tells a different story:

- **B** is the only near-the-money pool (WAC 6.84% vs. 7.30% market). Small loans refinance less when rates fall, and that protection is worth 6.8/32 at the cohort OAS. The market charges 14/32. The feature is real; the price is too high. Refinancing is 24% of its projected prepayments, against 6% for the 2021 pools.
- **C**'s California market is healthier (turnover index +0.20), but its larger loans and slightly slow history (credibility multiplier 0.97) leave it no faster than the cohort; its pay-up buys nothing.
- **F** is the important one. Florida's turnover index is −1.10 and the pool has run slow (multiplier 0.93), so it projects 4.45 CPR against 5.0 for its cohort. For a discount pool, slow is bad: it is worth about 12/32 *less* than TBA. A holder should sell it as a specified pool at its +2/32 if anyone will pay that, or deliver it into a TBA trade, where it is the kind of pool sellers prefer to deliver.

**How to read the edges.** The pool-specific speed multipliers carry roughly $\pm 12\%$ uncertainty, which is about $\pm 12$ bp of OAS for a discount pool. None of these edges is large against that, so the ranking is a tilt, not a set of trades to size up. Edges grow more reliable as pools season (credibility $Z$ rises) and as the regional signal persists.

### Projected CPR curves by pool and cohort

The base-case table in 5.4, extended to every pool and cohort, plotted as a fan: median CPR by month with the P10 to P90 band across rate paths. For Pool T the band is narrow in year one (4.5% to 5.0% at month 12) and widens with time (4.0% to 5.3% at month 120), because rates have more time to move.

### Turnover vs. refinance attribution

Per pool and cohort, the share of projected prepaid principal by cause, base case and under each rate scenario. For the 2021 discount cohort the answer is about four-fifths turnover in every scenario down to −200 bp. For a near-the-money pool like B it is two-thirds turnover today, but in a 100 bp rally refinancing jumps from 24% to 43% of prepaid principal and turnover falls to half.

### Regions and loan types driving turnover surprises

| Region | Turnover index (Sept) | Main driver of latest change | Credibility multiplier, pools in example | Effect on 12-mo CPR |
|---|---|---|---|---|
| Texas | −0.78 | Pending sales −1.1 sd | 1.07 (Pool T) | −0.13 CPR from the index |
| Florida | −1.10 | Pending sales and migration | 0.93 (Pool F) | about −0.2 CPR from the index |
| California | +0.20 | Job flows | 0.97 (Pool C) | about +0.03 CPR from the index |
| National | −0.25 | | 1.00 | |

(Texas computed in 5.3; the other indices are illustrative inputs.) By loan type, the decomposition from 5.2 shows where lock-in bites: loans with lock-in gaps above 3 points turn over at about 58% of their unlocked rate, versus about 80% at a gap of 1.5 points. Each month, the "news" from 5.3 (which indicator moved which region's index) explains the next month's surprises before the payoff data arrive.

### Rate-scenario sensitivity

The scenario table from 5.5 for every pool, plus effective duration and convexity, so a portfolio manager can see that Pool T gains about 6.0 points in a 100 bp rally while Pool B gains about 4.3, because refinancing (its 12-month CPR jumps from 6.8% to 11.9%) returns principal at par just when it is worth more than par.

### For other clients

- **Servicers:** the same projected hazards, applied to servicing-fee cash flows, value mortgage servicing rights, which lose value when loans prepay (the opposite sign to a discount MBS investor).
- **Brokerages:** the sale hazard summed by 3-digit zip (the finest geography in the agency data) gives a map of expected listings over the next year. Because the agency data are anonymized, this works only at aggregate geography. Targeting individual homeowners would need other data and falls under FCRA prescreening rules if it uses credit-bureau data.

---

## Symbol reference

| Symbol | Meaning | Step |
|---|---|---|
| $i$, $p$, $j$, $e$ | Loan, pool, region, payoff event | all |
| $t$, $h$, $\Delta$ | Month; forecast horizon in months; time step (1/12 year) | all |
| $c_i$, $c^{\text{pt}}$ | Loan note rate; pool pass-through coupon | 5.1, 5.2, 5.5 |
| $m_t$ | Market 30-year mortgage rate (PMMS) | 5.1, 5.2, 5.5 |
| $G_{i,t}$, $L_{i,t}$ | Rate incentive $c_i - m_{t-1}$; lock-in gap $\max(m_{t-1} - c_i, 0)$ | 5.1, 5.2 |
| $\mathcal{A}$, $\varepsilon$ | Anchor set of deep out-of-the-money payoffs; cash-out contamination share | 5.1 |
| $\text{qtr}_e$, $\text{drop}_e$ | Calendar quarter of payoff; recent-rate-drop flag | 5.1 |
| $f_S$, $f_R$ | Feature profiles of sale and refinance payoffs | 5.1 |
| $\pi$, $\omega_e$ | Sale share among ambiguous payoffs; posterior sale probability of payoff $e$ | 5.1 |
| $k \in \{0,S,R,D\}$ | Outcome: survive, sale, refinance, default | 5.2 |
| $h^{k}_{i,t}$, $\eta^{k}_{i,t}$ | Hazard of cause $k$; its linear score (log odds vs. survival) | 5.2 |
| $\mathbf{z}_{i,t}$, $\boldsymbol\beta^{k}$ | Loan-month features; coefficients for cause $k$ | 5.2 |
| $y^{k}_{i,t}$ | Outcome indicator (fractional for ambiguous payoffs) | 5.2 |
| $\text{age}_{i,t}$, $\text{CLTV}_{i,t}$, $\text{FICO}_i$, $b_{i,t}$, $\text{TX}_i$ | Loan age; current LTV; credit score; balance; Texas indicator | 5.2 |
| $\gamma_{\text{moy}}$ | Month-of-year (seasonality) effect | 5.2 |
| $\beta_f$ | Effect of the turnover index on sale odds | 5.2 |
| $f_t$, $f_{t\mid s}$ | Regional turnover factor; its estimate given data through $s$ | 5.3 |
| $x_{\ell,t}$, $\lambda_\ell$, $\psi_\ell$ | Standardized indicator; loading; noise variance | 5.3 |
| $\phi$, $q$ | Factor persistence; factor shock variance ($1 - \phi^2$) | 5.3 |
| $\Sigma_{t\mid s}$ | Variance of the factor estimate | 5.3 |
| $\tilde b_{i,t}$, $\kappa$ | Balance after scheduled principal; curtailment rate | 5.4 |
| SMM, CPR | Monthly and annualized prepayment rate | 5.4, 5.5 |
| $A_p$, $E_p$, $R_p$ | Actual and model-expected payoffs; their ratio | 5.4 |
| $n_p$, $K$, $Z_p$, $v$, $\bar h$ | Exposure in loan-months; credibility constant; credibility weight; between-pool variance of true multipliers; average monthly payoff probability | 5.4 |
| $\mu_p$ | Credibility-weighted pool speed multiplier | 5.4, 5.5 |
| $r_t$, $a$, $\sigma$, $\theta(t)$, $W_t$ | Short rate; mean reversion; volatility; curve-fitting drift; Brownian motion | 5.5 |
| $P(t,T)$, $F(0,t)$ | Zero-coupon bond price; instantaneous forward rate | 5.5 |
| $\alpha(t)$, $X_t$ | Deterministic and random parts of the short rate | 5.5 |
| $A(t,T)$, $B(t,T)$ | Hull-White bond-price functions | 5.5 |
| $y^{(10)}_t$, $s^{M}$ | 10-year par yield on a path; mortgage–Treasury spread | 5.5 |
| $n$, $N$ | Rate path; number of paths | 5.5 |
| $\text{CF}_{n,t}$, $B_{n,t}$ | Investor cash flow; pool balance on path $n$ | 5.5 |
| $\delta$, $\tau_t$ | Payment delay after month end; payment time in years | 5.5 |
| $s$, OAS, ZV | Spread over short rates; option-adjusted spread; zero-volatility spread | 5.5 |
| $\text{Px}$, $\text{Px}^{\text{mkt}}$ | Model price; market price (per \$100 of balance) | 5.5 |
| $k$, $k^*$ | Hazard scaling factor; market-implied value of it | 5.5 |
| $D^{\text{eff}}$, $C^{\text{eff}}$ | Effective duration; effective convexity | 5.5 |
| WAC, WALA, WAL, LTV, TBA | Weighted-average coupon; weighted-average loan age; weighted-average life; loan-to-value; to-be-announced market | all |
