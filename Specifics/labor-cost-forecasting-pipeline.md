# Service 1: Labor Cost Forecasting, explained as a pipeline

*Market backdrop: mid-October 2026. All lender-level numbers below are one worked example run end to end through a simulation; they are illustrative, not figures from any real lender.*

---

## 0. What we are forecasting, and why it needs five models

### The object being forecast

A mortgage lender's production business has a simple economic shape. Every loan it closes earns revenue once, when the loan is sold to an investor (Fannie Mae, Freddie Mac, Ginnie Mae, or a private buyer). To get there, the lender pays people: **loan officers** who take applications, **processors** who gather documents, **underwriters** who decide whether the loan meets the investor's rules, and **closers** who prepare the final documents. Most of those people are salaried, so their cost does not fall when volume falls.

The quantity this service forecasts, for a lender (or for a group of lenders), is the **production cost per loan** (CPL): total production expense in a quarter divided by the number of loans closed in that quarter. We forecast it quarter by quarter for the next year, as a full distribution (P10/P50/P90, where P10 is the 10th percentile: a value the outcome falls below 10% of the time), together with the two things a lender can act on: the **breakeven volume** (how many closings are needed to cover costs) and the **headcount** that fits the forecast workload.

Note that "cost function" here means the economist's object, the relationship between cost and output. It is not a machine-learning loss function.

### Why it is worth forecasting now

- **Volume is weak and was just knocked down again.** The 30-year fixed rate (Freddie Mac PMMS, the Primary Mortgage Market Survey, a weekly national average) is near 7.3% after spiking above 7% in late September on the Iran-conflict oil shock, with the 10-year Treasury pushed toward 5%. The Fed hiked to 3.75–4.00% on September 16.
- **Refinancing is mostly switched off.** About half of outstanding mortgages carry rates below 4% and only about 22% are above 6%. A borrower at 3% has no reason to refinance at 7.3%; this is the **lock-in effect**. Only the thin slice of recent high-rate borrowers can respond to a rate dip, so refinance volume is small but very sensitive to rates near current levels.
- **Costs are at a record.** Origination cost is about \$11,900 per loan against a long-run average near \$7,900, and about two-thirds of it is compensation.
- **The mechanism is the fixed-cost trap.** Staffing adjusts slowly while volume swings fast. When volume drops, the same payroll is divided over fewer loans, and cost per loan spikes. Lenders that understand *how much* and *for how long* can staff ahead of the cycle instead of behind it.

### Why five models instead of one

A single regression of cost per loan on interest rates would mix together four different mechanisms that move on different clocks:

$$\text{CPL}_q = \underbrace{\frac{\overbrace{H_q \bar c_q}^{\text{payroll (1.4)}} + K}{\underbrace{C_q}_{\text{closings (1.1, 1.3)}}}}_{\text{fixed cost spread over volume}} + \underbrace{v + \gamma s_q}_{\text{per-loan variable cost (1.2)}}$$

Applications respond to rates within weeks. Closings follow applications with a lag of one to three months, and only a fraction of applications ever close. Headcount follows workload with a lag of several quarters. Cost per loan is then an accounting combination of all three. Each piece is modeled with the tool that fits it, and the rate scenarios from 1.5 are pushed through the whole chain.

| Step | Model | Produces |
|---|---|---|
| 1.1 | Regression with AR(1) errors (purchase) + structural S-curve (refinance) | Monthly applications $P_t$, $R_t$, $A_t$ |
| 1.3 | Logistic regression on HMDA + lag distribution | Pull-through $\rho_t$ and monthly closings $C_t$ |
| 1.4 | Error-correction model (Engle–Granger) | Quarterly headcount $H_q$ and payroll $F^{\text{pers}}_q$ |
| 1.2 | Cost-per-loan equation with partial pooling | $\text{CPL}_q$ and breakeven volume $C^*_q$ |
| 1.5 | Rate scenario simulator + Monte Carlo + peer benchmarking | Fan chart, $\Pr(\text{loss})$, staffing range, scale vs. efficiency diagnosis |

### The actual flow of numbers (and why the numbering is out of order)

The step numbers are labels, not the order of computation. The numbers actually flow like this:

```
1.5 rate path m_t ──► 1.1 purchase apps P_t ──┐
         │                                     ├─► A_t ──► 1.3 pull-through ρ_t ──► closings C_t ────────────┐
         └─────────► 1.1 refi apps R_t ────────┘    │                                                        │
                                                    └──► workload W_q ──► 1.4 headcount H_q ──► payroll F_q ──┤
                                                                                                              ▼
                                                                       1.2 cost per loan CPL_q, breakeven C*_q
                                                                                                              │
                                        1.5 Monte Carlo: repeat for 20,000 rate paths ◄───────────────────────┘
                                                    │
                                                    ▼
                          fan chart, Pr(loss), staffing range, peer benchmarking ──► report
```

So 1.5 appears twice: at the start, it supplies a rate path; at the end, it repeats the whole chain over thousands of rate paths and summarizes. The sections below follow the flow: **1.1 → 1.3 → 1.4 → 1.2 → 1.5**. In 1.1 through 1.2 the worked example uses the **central rate path** (the market's forward curve with no shocks); 1.5 then explains where that path comes from and replaces it with simulated paths.

Two links deserve emphasis because they are easy to miss. First, **applications create the work but only closings bring revenue**: workload in 1.4 is built from applications, while the denominator in 1.2 is closings. That gap, and the lag between the two, is why cost per loan spikes *after* a volume drop rather than with it. Second, the **same rate path** drives both purchase and refinance applications *and* changes how many applications close (1.3), so all of them must be simulated on one shared path in 1.5, not separately.

### Timeline used in the worked example

- **Today:** mid-October 2026. The last complete month of data is **September 2026**; the last complete quarter is **Q3 2026**.
- **October's rate is treated as known** at 7.30%: half the month is already in, and October is where every simulated rate path starts.
- **Forecast horizon:** monthly from **October 2026 ($h = 1$) through September 2027 ($h = 12$)**, summarized into four quarters: **Q4 2026, Q1 2027, Q2 2027, Q3 2027**. Here $h$ is the forecast horizon in months.

### The running example: Lender L

A mid-size **independent mortgage bank** (a non-bank lender that funds loans on a warehouse credit line and sells them shortly after closing).

- **Q3 2026 actuals:** about 3,640 closings (roughly 1,215 a month), average loan about \$340,000, about 29% of closings **government loans** (FHA, insured by the Federal Housing Administration, and VA, guaranteed by the Department of Veterans Affairs).
- **Salaried production and support staff:** **540** people at the end of Q3 2026, at a fully loaded cost (salary, benefits, payroll taxes) of **\$34,000 per person per quarter**, rising with wage inflation of 0.85% a quarter (about 3.4% a year, in line with CPI).
- **Other fixed costs** (technology licenses, occupancy, compliance): **\$4.4 million per quarter**.
- **Per-loan variable cost:** about **\$4,600** (loan officer commission about \$2,700, plus per-loan third-party and technology fees), plus about **\$1,050 extra** for a government loan.
- **Revenue per closed loan:** **\$12,300** (gain on sale plus origination fees, about 360 basis points of the loan amount).
- **Refinance catchment:** the pool of existing mortgages from which L realistically draws refinance applications (its own servicing book plus its historical share of the wider market), about **200,000 loans**.
- **Monthly applications in 2026:**

| Month 2026 | Mar | Apr | May | Jun | Jul | Aug | Sep |
|---|---|---|---|---|---|---|---|
| Purchase $P_t$ | 1,200 | 1,240 | 1,250 | 1,180 | 1,150 | 1,120 | 940 |
| Refinance $R_t$ | 480 | 520 | 610 | 700 | 690 | 600 | 470 |
| Average 30-yr rate $m_t$ (%) | 6.45 | 6.40 | 6.35 | 6.30 | 6.35 | 6.45 | 6.80 |

The summer dip toward 6.3% produced a small refinance wave; the late-September spike ended it.

The punchline we will derive: L's cost per loan rises from **\$11,500 in Q3 2026 to about \$12,500 in Q4 and about \$13,500 in Q1 2027** at the median, and the full-year figure is about **\$12,700** (P10 \$12,100, P90 \$13,160). If L adjusts staff the way lenders historically have, it has about an **81% chance of losing money** on production over the next four quarters. About **70% of the uncertainty is interest rates**. Cutting to about 455 people by year-end through attrition brings the loss probability down to about 42%.

---

## Step 1.1: Volume forecasting (regression with AR(1) errors + refinance S-curve)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Lender's monthly purchase applications, 10 years of history | $P_t$ | Lender's loan origination system (LOS); for a market-level version, HMDA (the Home Mortgage Disclosure Act loan-level data) or the MBA application survey |
| Month-level drivers: the 30-year mortgage rate and calendar-month indicators | $\mathbf{x}_t$ | PMMS for history; **rate path $m_t$ from 1.5** for the forecast |
| Distribution of the catchment's outstanding loans across note-rate buckets | $N_t$, $w_{k,t}$ | FHFA National Mortgage Database (NMDB) for the market; the lender's servicing tape for its own book |
| Refinance S-curve parameters | $\pi_0$, $\pi_{\max}$, $I_0$, $\kappa$ | Fitted on agency loan-level data (Fannie Mae / Freddie Mac) |

**Terms.** A **purchase application** is a loan request to buy a home; a **refinance application** replaces an existing mortgage with a new one. The **note rate** $c$ is the interest rate written on an existing mortgage. The **refinance incentive** is how far the note rate is above today's market rate: a borrower at 7.75% facing a 7.30% market has an incentive of 0.45 percentage points (pp). **Burnout** is the tendency of a pool of loans to become less responsive to rate drops over time, because the borrowers most willing to refinance have already done so and left.

### Process, part 1: purchase applications

Purchase demand responds to rates, but also has momentum: a soft month tends to be followed by another soft month for reasons the drivers do not capture (buyer sentiment, local inventory). A regression with **autocorrelated errors** captures both. Let $y_t = \log P_t$. Then

$$y_t = \mathbf{x}_t'\boldsymbol\beta + \eta_t, \qquad \eta_t = \phi\, \eta_{t-1} + \varepsilon_t, \qquad \varepsilon_t \sim \mathcal{N}(0, \sigma_\varepsilon^2).$$

- $\mathbf{x}_t$ is the vector of month-$t$ drivers: a constant, the mortgage rate $m_t$, and eleven calendar-month indicators (each equal to 1 in its month and 0 otherwise). We write month-level drivers as $\mathbf{x}_t$ to keep them distinct from the application-level features $\mathbf{a}_i$ in 1.3.
- $\boldsymbol\beta$ holds the coefficients. The coefficient on $m_t$, written $\beta_m$, is a **semi-elasticity**: a 1 pp rise in the rate changes purchase applications by about $100\beta_m$ percent.
- $\eta_t$ is the regression error, which is not independent across months: it follows an **AR(1)** process (first-order autoregression), so this month's error is $\phi$ times last month's plus a fresh shock $\varepsilon_t$. The parameter $\phi$, between 0 and 1 here, measures how persistent surprises are.

**Why not ordinary least squares?** OLS would give an unbiased $\boldsymbol\beta$ but understate its standard errors (it treats 120 correlated months as 120 independent pieces of evidence), and, more importantly for forecasting, it throws away $\phi$. Knowing that September came in below the model is information about October.

**Estimation, Cochrane–Orcutt.** The two pieces are estimated jointly by iterating:

1. Fit OLS of $y_t$ on $\mathbf{x}_t$; keep the residuals $\hat\eta_t$.
2. Regress $\hat\eta_t$ on $\hat\eta_{t-1}$ to get $\hat\phi$.
3. **Quasi-difference** the data, $y_t - \hat\phi y_{t-1}$ and $\mathbf{x}_t - \hat\phi \mathbf{x}_{t-1}$. Because $\eta_t - \phi\eta_{t-1} = \varepsilon_t$, the transformed regression has independent errors, so OLS on it is efficient. Refit $\boldsymbol\beta$.
4. Recompute residuals from the original equation and repeat 2–3 until $\hat\phi$ stops changing.

Maximum likelihood on the same model gives essentially the same answer and is the alternative when you want exact standard errors.

**Forecast.** At the forecast origin $T$ (September 2026), the last error $\hat\eta_T = y_T - \mathbf{x}_T'\hat{\boldsymbol\beta}$ is known. Each month ahead, its expected value shrinks by a factor $\hat\phi$:

$$\hat y_{T+h} = \mathbf{x}_{T+h}'\hat{\boldsymbol\beta} + \hat\phi^{\,h}\,\hat\eta_T, \qquad \operatorname{Var}\big(y_{T+h} - \hat y_{T+h}\big) = \sigma_\varepsilon^2 \sum_{j=0}^{h-1} \hat\phi^{\,2j}.$$

The forecast of $P_{T+h}$ itself is $e^{\hat y_{T+h}}$, which is the **median**; the mean is larger by the factor $e^{\operatorname{Var}/2}$ (about 0.15% here, negligible). Note that $\mathbf{x}_{T+h}$ contains the **future** rate $m_{T+h}$, which is why this step needs a rate path from 1.5.

### Process, part 2: refinance applications

Refinancing is not a smooth function of the rate level; it depends on how many existing borrowers are **in the money**. So refinance applications are built structurally, bucket by bucket. Divide the catchment's $N_t$ loans into note-rate buckets $k$ with representative note rate $c_k$ and share $w_{k,t}$ (so $\sum_k w_{k,t} = 1$). Each bucket's incentive is

$$I_{k,t} = c_k - m_t,$$

and the probability that a loan in the bucket applies to refinance this month follows an **S-curve** (a logistic function of the incentive):

$$\pi(I) = \pi_0 + \frac{\pi_{\max}}{1 + e^{-(I - I_0)/\kappa}}.$$

- $\pi_{\max}$ is the extra monthly application probability for a deeply in-the-money loan.
- $I_0$ is the incentive at which the S-curve is halfway up; borrowers need some incentive just to cover closing costs.
- $\kappa$ is the scale: how quickly the curve rises around $I_0$ (the project context writes this as $s$; we use $\kappa$ to keep $s$ free for the loan-mix share in 1.2).
- $\pi_0$ is a floor we add to the context's formula. It represents **cash-out refinances** and life-event refinances (divorce, removing a co-borrower) that happen at any rate. In a 7% market these are most of the refinance business, and without the floor the model would predict almost no refinance applications.

Refinance applications are then

$$R_t = N_t \sum_k w_{k,t}\,\pi(I_{k,t}).$$

**Burnout.** Each month, the loans that refinance leave their old bucket and re-enter the catchment as a new bucket with note rate $m_t$ (they now have a fresh loan at today's rate, so zero incentive). With $\rho^{\text{refi}}_t$ the share of refinance applications that close (from 1.3),

$$N_{t+1} w_{k,t+1} = N_t w_{k,t}\big(1 - \rho^{\text{refi}}_t\,\pi(I_{k,t})\big), \qquad \text{new bucket: } c = m_t,\ \text{size } \rho^{\text{refi}}_t R_t.$$

This is how bucket shares update each month. It matters only when rates fall far enough to drain the high-rate buckets.

**Total applications.** $A_t = P_t + R_t$.

### Worked example

**Fitting the purchase model.** Cochrane–Orcutt on L's 120 months of purchase applications (October 2016 to September 2026) gives

$$\hat\beta_m = -0.089 \;(\text{s.e. } 0.008), \qquad \hat\phi = 0.61, \qquad \hat\sigma_\varepsilon = 0.054.$$

A 1 pp rise in the rate cuts purchase applications by about 9%. Plain OLS gives almost the same slope ($-0.086$) but a standard error of 0.004, half the correct one: it is overconfident because it ignores the autocorrelation. The fitted seasonal pattern, expressed as log deviations from an average month (centered so they average to zero), with intercept $\hat\beta_0 = 7.541$:

| Month | Jan | Feb | Mar | Apr | May | Jun | Jul | Aug | Sep | Oct | Nov | Dec |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Seasonal factor | −0.04 | 0.00 | +0.09 | +0.08 | +0.12 | +0.09 | +0.04 | +0.03 | −0.08 | −0.03 | −0.09 | −0.20 |

December runs about 18% below an average month ($e^{-0.20} = 0.82$); spring runs about 10% above.

**The September residual.** September's rate averaged 6.80%. The model's fitted value:

$$\mathbf{x}_T'\hat{\boldsymbol\beta} = 7.541 - 0.089 \times 6.80 - 0.08 = 6.8558, \qquad \hat\eta_T = \log 940 - 6.8558 = 6.8459 - 6.8558 = -0.0099.$$

September came in about 1% below what the rate and the season explain: a small negative surprise.

**October and November forecasts** (central rate path: 7.300% in October, 7.282% in November):

$$\hat y_{\text{Oct}} = 7.541 - 0.089(7.300) - 0.03 + 0.61(-0.0099) = 6.8553, \qquad \hat P_{\text{Oct}} = e^{6.8553} = 949,$$

$$\hat y_{\text{Nov}} = 7.541 - 0.089(7.282) - 0.09 + 0.61^2(-0.0099) = 6.7992, \qquad \hat P_{\text{Nov}} = 897.$$

The September surprise has almost no effect by November ($0.61^2 = 0.37$ of an already small number). The forecast uncertainty grows with the horizon: for October the log standard deviation is 0.054, giving an 80% interval of about 885 to 1,017; for September 2027 it is $0.054\sqrt{\sum_{j=0}^{11} 0.61^{2j}} = 0.068$, and that is *before* adding rate uncertainty, which 1.5 handles.

**Refinance applications in October.** The catchment of 200,000 loans, by note-rate bucket, with S-curve parameters $\pi_0 = 0.0012$, $\pi_{\max} = 0.035$, $I_0 = 0.60$ pp, $\kappa = 0.25$ pp:

| Bucket note rate $c_k$ | 2.75 | 3.50 | 4.50 | 5.50 | 6.25 | 6.75 | 7.25 | 7.75 |
|---|---|---|---|---|---|---|---|---|
| Loans $N w_k$ | 42,000 | 60,000 | 34,000 | 24,000 | 18,000 | 12,000 | 6,000 | 4,000 |
| Incentive at $m = 7.30$ | −4.55 | −3.80 | −2.80 | −1.80 | −1.05 | −0.55 | −0.05 | +0.45 |
| Monthly $\pi(I)$ | .00120 | .00120 | .00120 | .00120 | .00125 | .00155 | .00362 | .01360 |
| Applications | 50.4 | 72.0 | 40.8 | 28.9 | 22.5 | 18.6 | 21.7 | 54.4 |

Check one cell, the 7.75% bucket: $(I - I_0)/\kappa = (0.45 - 0.60)/0.25 = -0.6$, so $\pi = 0.0012 + 0.035/(1 + e^{0.6}) = 0.0012 + 0.0124 = 0.0136$, and $4{,}000 \times 0.0136 = 54.4$.

Total: $R_{\text{Oct}} = 309$, of which 240 ($200{,}000 \times 0.0012$) is the cash-out floor. Total applications $A_{\text{Oct}} = 949 + 309 = 1{,}258$.

**The S-curve is what makes refinance volume asymmetric.** Rerun the same table at other rates:

| Rate $m$ | 8.00% | 7.30% | 6.80% (September) | 6.30% (summer low) |
|---|---|---|---|---|
| Refinance applications per month | 246 | 309 | 462 | 740 |

A 0.7 pp *rise* costs L about 60 applications a month; a 1.0 pp *fall* adds about 430. That convexity is invisible to a linear model and is the main reason the cost distribution in 1.5 has a long favorable tail. (The 462 at 6.80% is close to the 470 actually observed in September, a useful sanity check.)

**Burnout, numerically.** Hold the rate at 6.30% for six months. Without burnout, refinance applications stay at 740 a month. With burnout they drift down to 739, 734, 729, 723, 718, 713: about 4% lower after six months, because the 7.25% and 7.75% buckets are being drained. In this simple bucket model burnout is modest; in real data it is stronger, because within each bucket the most rate-sensitive borrowers leave first. A richer version adds a "times previously in the money" variable to $\pi$.

**The central path.** Running both models month by month along the central rate path (the October to September values of $m_t$ are listed in 1.5):

| | Oct | Nov | Dec | Jan | Feb | Mar | Apr | May | Jun | Jul | Aug | Sep |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| $P_t$ | 949 | 897 | 806 | 948 | 989 | 1,084 | 1,076 | 1,121 | 1,090 | 1,039 | 1,030 | 924 |
| $R_t$ | 309 | 313 | 317 | 321 | 325 | 330 | 334 | 338 | 343 | 348 | 353 | 358 |

Refinance applications creep up only because the forward curve drifts down 0.2 pp over the year.

### Output

Monthly $P_t$, $R_t$ and $A_t$ for $h = 1, \dots, 12$, on any given rate path, plus the sampling distributions of the shocks ($\varepsilon_t$ for purchase; a multiplicative noise $\nu_t$ with standard deviation 0.10 for refinance, estimated from the S-curve's monthly residuals) so that 1.5 can draw model error as well as rate paths.

### Hand-off

$P_t$ and $R_t$ go to 1.3 (pull-through depends on the purchase/refinance mix) and to 1.4 (they define workload). The refinance step's burnout update takes the refinance pull-through $\rho^{\text{refi}}_t$ back from 1.3.

### Validation

Out-of-time backtest: fit through month $t$, forecast $t+1$ to $t+12$ using the rates that actually happened (to isolate this step's error from rate-forecast error), compare with actual applications. Check that about 80% of actual months fall inside the P10 to P90 band. For the S-curve, plot actual monthly application rates by bucket against incentive; the fitted curve should run through the cloud at every incentive level, especially near $I_0$.

---

## Step 1.3: Pull-through and closings (logistic regression on HMDA + lag distribution)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Application-level records: purpose, loan type, debt-to-income, loan-to-value, action taken, application and action dates | $\mathbf{a}_i$, outcome | HMDA loan-level data (public, annual); the lender's own LOS pipeline data for its own version |
| Rate change in the month after application | $\Delta m_i$ | PMMS matched to application date |
| Monthly purchase and refinance applications | $P_t$, $R_t$ | Step 1.1 |
| Composition of current applications by segment | | Lender's pipeline report |

**Terms.** **Pull-through** is the share of applications that end up as closed (originated) loans. The rest is **fallout**: applications withdrawn by the borrower, denied by the underwriter, or closed for incompleteness. **DTI** (debt-to-income) is the borrower's total monthly debt payments divided by gross monthly income. **LTV** (loan-to-value) is the loan amount divided by the property value. The **action date** in HMDA is the date the application reached its final outcome.

### Process

**Application-level model.** For application $i$, with feature vector $\mathbf{a}_i$ (an intercept; indicators for refinance, government loan, DTI above 43%, LTV above 90%; and the rate change $\Delta m_i$ interacted with purpose), the probability it closes is a logistic regression:

$$p_i = \Pr(\text{application } i \text{ originates}) = \frac{1}{1 + e^{-\mathbf{a}_i'\boldsymbol\theta}}.$$

$\boldsymbol\theta$ is estimated by maximum likelihood on HMDA, coding "loan originated" as 1 and denied, withdrawn, or closed for incompleteness as 0. (Applications approved but not accepted are coded 0 as well; they consumed labor and did not close.) Each coefficient is a change in the **log odds** $\log\big(p/(1-p)\big)$.

**Why a rate-change feature.** Pull-through depends on what rates do *after* the application. If rates fall during processing, refinance borrowers often abandon the application and reapply elsewhere at the lower rate, so refinance pull-through falls. If rates rise, purchase borrowers who were near the DTI limit stop qualifying, so purchase pull-through falls, while refinance borrowers lock in and close because they have no better option. Letting $\Delta m_i$ enter with a different sign by purpose makes pull-through **depend on the rate path**, which matters in 1.5.

**Segment rates.** For a segment $s$ (purchase or refinance), the segment pull-through is the average predicted probability over that segment's applications:

$$\rho_s = \frac{1}{n_s}\sum_{i \in s} p_i.$$

On the training data this equals the observed rate exactly (a property of logistic regression with an intercept and segment indicators), so the model adds nothing *if the mix never changes*. Its value is that it recomputes $\rho_s$ when composition shifts. If the share of high-DTI government purchase loans rises, as it does when affordability worsens, the model lowers $\rho_{\text{purch}}$ even though no such month exists in the training data.

**Monthly blend.** Since purchase and refinance applications close at different rates,

$$\rho_t = \frac{P_t}{A_t}\,\rho_{\text{purch},t} + \frac{R_t}{A_t}\,\rho_{\text{refi},t},$$

where the subscript $t$ on the segment rates records that they depend on $\Delta m_t = m_{t+1} - m_t$, the rate change in the month after cohort $t$ applied. A **cohort** is the set of applications taken in one month.

**From applications to closings.** An application taken in month $t$ closes in month $t$, $t+1$, $t+2$, and so on. Let $\lambda_\ell$ be the share of eventual closings that happen $\ell$ months after application, measured from the gap between application and action dates in HMDA (so $\sum_\ell \lambda_\ell = 1$). Closings in month $t$ collect the closing cohorts of the last several months:

$$C_t = \sum_{\ell=0}^{4} \lambda_\ell\, \rho_{t-\ell}\, A_{t-\ell}.$$

This convolution is what makes closings **lag** applications.

### Worked example

**Fitting the logistic model.** On a sample of 60,000 recent applications:

| Feature | Intercept | Refinance | Government | DTI > 43% | LTV > 90% | $\Delta m$ × purchase | $\Delta m$ × refinance |
|---|---|---|---|---|---|---|---|
| $\hat\theta$ | 1.25 | −0.76 | −0.35 | −0.43 | −0.18 | −0.49 | +0.82 |
| s.e. | 0.017 | 0.019 | 0.019 | 0.020 | 0.021 | 0.057 | 0.071 |

**One application.** A government purchase loan with DTI above 43% and LTV above 90%, with rates flat after application:

$$\mathbf{a}_i'\hat{\boldsymbol\theta} = 1.25 - 0.35 - 0.43 - 0.18 = 0.29, \qquad p_i = \frac{1}{1 + e^{-0.29}} = 0.572.$$

A conventional purchase with no risk flags has $p = 1/(1 + e^{-1.25}) = 0.777$. The risk flags cost about 20 points of pull-through.

**Segment rates at L's current composition.** L's purchase applications are 35% government, 25% high-DTI, 40% high-LTV; its refinance applications are 25%, 20% and 10%. Averaging $p_i$ over that mix with $\Delta m = 0$:

$$\rho_{\text{purch}} = 0.718, \qquad \rho_{\text{refi}} = 0.573.$$

These match the observed training averages (0.716 and 0.571) to within rounding of the coefficients, as the theory says they should.

**Rate-path dependence.** Holding composition fixed:

| Rate change in the month after application | −0.50 pp | 0 | +0.50 pp |
|---|---|---|---|
| $\rho_{\text{purch}}$ | 0.764 | 0.718 | 0.666 |
| $\rho_{\text{refi}}$ | 0.473 | 0.573 | 0.668 |

The September cohort is a live example: it applied at 6.80% and then rates jumped 0.5 pp to 7.30%. Its purchase pull-through falls to 0.666, but its refinance pull-through *rises* to 0.668, because those refinance borrowers locked before the spike and now have nowhere better to go.

**October's blended pull-through.** On the central path, $\Delta m_{\text{Oct}} = 7.282 - 7.300 = -0.018$, so $\rho_{\text{purch}} = 0.7195$ and $\rho_{\text{refi}} = 0.5698$:

$$\rho_{\text{Oct}} = \frac{949}{1{,}258}(0.7195) + \frac{309}{1{,}258}(0.5698) = 0.6827.$$

**Lag distribution** (from HMDA application-to-action gaps): $\lambda_0 = 0.05$, $\lambda_1 = 0.50$, $\lambda_2 = 0.33$, $\lambda_3 = 0.10$, $\lambda_4 = 0.02$. Half of all closings happen the month after application; very few take more than three months.

**October closings:**

| Cohort | Lag $\ell$ | Applications $A$ | Pull-through $\rho$ | Eventual closings $\rho A$ | $\lambda_\ell$ | Closes in October |
|---|---|---|---|---|---|---|
| Oct 2026 | 0 | 1,258 | 0.683 | 858.9 | 0.05 | 42.9 |
| Sep 2026 | 1 | 1,410 | 0.667 | 940.5 | 0.50 | 470.2 |
| Aug 2026 | 2 | 1,720 | 0.668 | 1,148.6 | 0.33 | 379.0 |
| Jul 2026 | 3 | 1,840 | 0.665 | 1,223.2 | 0.10 | 122.3 |
| Jun 2026 | 4 | 1,880 | 0.665 | 1,249.3 | 0.02 | 25.0 |
| **Total** | | | | | | **1,039.5** |

Most of October's closings come from August and September applications, which are already known. That is why the first quarter of any forecast is the most certain one.

**Quarterly closings on the central path:**

| Quarter | Q3 2026 (actual) | Q4 2026 | Q1 2027 | Q2 2027 | Q3 2027 |
|---|---|---|---|---|---|
| Applications | 4,970 | 3,591 | 3,998 | 4,302 | 4,051 |
| Closings $C_q$ | 3,644 | 2,810 | 2,503 | 2,870 | 2,890 |
| Government share of closings $s_q$ | 29.4% | 29.8% | 30.4% | 30.7% | 30.6% |

Notice the timing: applications bottom out in Q4 (December is the weakest month), but **closings bottom out in Q1**, because December and January applications close in January and February. Applications are already recovering in Q1, so the work is rising just as revenue hits its low.

### Output

Monthly closings $C_t$ on any rate path; segment pull-through $\rho_{s,t}$; the government share of closings $s_q$ (needed in 1.2); and fallout rates by segment for the report.

### Hand-off

$C_t$, summed to quarters, is the denominator of cost per loan in 1.2. $s_q$ goes to 1.2. $\rho^{\text{refi}}_t$ goes back to 1.1 for the burnout update.

### Validation

Calibration on out-of-time data (train on 2023–2024 HMDA, test on 2025): group applications by predicted $p_i$ in bins and check that the observed origination rate in each bin matches. Check the lag distribution's stability across years: processing times stretch in booms, so $\lambda_\ell$ estimated in a slow year will understate lags in a busy one.

---

## Step 1.4: Staffing lag (error-correction model)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Quarterly industry headcount | $H_q$ | BLS Current Employment Statistics for NAICS 522292 (Real Estate Credit), the closest public proxy for mortgage-lending employment |
| Quarterly workload | $W_q$ | Built from applications (history: HMDA or MBA counts; forecast: **1.1**) |
| Lender's own headcount and loaded cost per head | $H_{L,q}$, $\bar c_q$ | Lender's HR and payroll records |

**Terms.** **Headcount** here means salaried production and support staff, not commission-only loan officers (their pay is per loan and sits in the variable cost $v$ in 1.2). **Workload** is a measure of how much processing work comes in. **Cointegration** means two series that each wander without a fixed mean (they are **non-stationary**) are tied together in the long run, so a particular combination of them is stationary. **Stationary** means the series fluctuates around a fixed mean with a fixed spread.

### Why an error-correction model

Lenders do not set headcount to match this quarter's workload. Hiring takes time, layoffs are costly and painful, and managers wait to see whether a drop is temporary. But headcount does not drift away from workload forever either: in the long run there is an equilibrium staffing level for a given workload. An **error-correction model** (ECM) describes exactly that: a long-run target, plus a speed at which actual headcount closes the gap.

### Process

**Workload.** Define

$$W_q = \sum_{t \in q} \big(\omega_P P_t + \omega_R R_t\big),$$

with **labor weights** $\omega_P = 1$ and $\omega_R = 0.7$, because a refinance file takes less work than a purchase (no purchase contract, no moving closing date, simpler appraisal). Setting $\omega_R = \omega_P = 1$ gives the unweighted version $W_q = \sum_{t \in q} A_t$. Whichever definition is chosen must be used **identically** in the historical estimation and in the forecast, or the estimated coefficients will not apply.

**Long-run relationship (step 1 of Engle–Granger).**

$$\log H_q = a_H + b \log W_q + u_q.$$

$b$ is the long-run **elasticity** of headcount to workload: 1% more workload requires $b$% more staff in equilibrium. $b < 1$ means economies of scale. $u_q$ is the **disequilibrium**: positive when the lender is overstaffed relative to its workload.

Estimate by OLS, then test whether $\hat u_q$ is stationary with an **ADF test** (augmented Dickey–Fuller: regress $\Delta \hat u_q$ on $\hat u_{q-1}$ and a few lags of $\Delta \hat u_q$; a sufficiently negative $t$-statistic on $\hat u_{q-1}$ rejects "no cointegration"). Because $\hat u$ comes from an estimated regression, the critical values are the Engle–Granger ones, which are more negative than the standard Dickey–Fuller values. If cointegration is rejected, there is no stable long-run target and an ECM is the wrong model.

**Short-run dynamics (step 2).**

$$\Delta \log H_q = \alpha\, \hat u_{q-1} + \delta\, \Delta \log W_q + \zeta_q, \qquad \zeta_q \sim \mathcal{N}(0, \sigma_H^2).$$

- $\alpha < 0$ is the **speed of adjustment**: each quarter, a fraction $-\alpha$ of last quarter's gap is closed. The **half-life** of a staffing gap is $\ln 0.5 / \ln(1 + \alpha)$ quarters.
- $\delta$ is the **immediate response**: the share of this quarter's workload change that shows up in headcount right away.
- $\zeta_q$ is the quarter's unexplained hiring or layoff shock.

**Applying industry dynamics to one lender.** The speeds $\alpha$, $\delta$ and the elasticity $b$ are estimated on 27 years of industry data, because one lender's history is too short. The lender gets its own intercept $a_L$, calibrated so that $\exp(a_L + b \log W)$ matches its own equilibrium headcount (from its capacity analysis: how many files per processor, underwriter and closer it can carry at normal utilization).

**Payroll.** $F^{\text{pers}}_q = H_q\, \bar c_q$, where $\bar c_q$ is the fully loaded cost per salaried employee for the quarter.

### Worked example

**Estimation on industry data** (2000 Q1 to 2026 Q3, 107 quarters):

$$\log H_q = 2.28 + 0.653 \log W_q + \hat u_q.$$

ADF on $\hat u_q$: $-4.49$, against Engle–Granger critical values of $-3.39$ (5%) and $-4.00$ (1%), so cointegration is accepted at the 1% level. The ECM:

$$\Delta \log H_q = -0.154\,\hat u_{q-1} + 0.280\,\Delta \log W_q + \hat\zeta_q, \qquad \text{s.e. } 0.016 \text{ and } 0.017, \quad \hat\sigma_H = 0.012.$$

We use $b = 0.65$, $\alpha = -0.15$, $\delta = 0.28$. In words: in equilibrium, a lender with 10% more workload needs about 6.5% more staff; when workload changes, 28% of the change in staff happens the same quarter; after that, 15% of the remaining gap closes each quarter, a half-life of $\ln 0.5 / \ln 0.85 = 4.3$ quarters. Staffing takes about a year to close half of any gap.

(A practical caution: in a short sample with slow adjustment, the static levels regression can understate $b$, because headcount spends long stretches away from equilibrium. If the estimate looks implausible, cross-check with dynamic OLS, which adds leads and lags of $\Delta \log W$ to the levels regression.)

**Calibrating Lender L.** Q3 2026 workload, from the application table:

$$W_{\text{Q3}} = (1{,}150 + 1{,}120 + 940) + 0.7\,(690 + 600 + 470) = 3{,}210 + 1{,}232 = 4{,}442.$$

L's capacity analysis says 515 salaried staff is the right level for that workload, so

$$a_L = \ln 515 - 0.65 \ln 4{,}442 = 6.2442 - 5.4593 = 0.7849.$$

L actually has 540, so it enters Q4 overstaffed: $u_{\text{Q3}} = \ln(540/515) = 0.0474$, about 5%.

**Q4 2026.** Workload from 1.1's central path: $W_{\text{Q4}} = 2{,}652 + 0.7 \times 939 = 3{,}310$, a fall of $\Delta \log W = \ln(3{,}310/4{,}442) = -0.2943$ (about 25%).

$$\Delta \log H_{\text{Q4}} = -0.15(0.0474) + 0.28(-0.2943) = -0.0071 - 0.0824 = -0.0895,$$

$$H_{\text{Q4}} = 540\, e^{-0.0895} = 493.8.$$

The equilibrium for that workload is $H^*_{\text{Q4}} = \exp(0.7849 + 0.65 \ln 3{,}310) = 425.3$. So even after shedding 46 people, L ends Q4 with about 68 more staff than its workload calls for: an **overhang** of 16%.

**Q1 2027.** Workload recovers seasonally to 3,705 ($\Delta \log W = +0.1129$). The gap entering the quarter is $u_{\text{Q4}} = \ln(493.8/425.3) = 0.1492$:

$$\Delta \log H_{\text{Q1}} = -0.15(0.1492) + 0.28(0.1129) = -0.0224 + 0.0316 = +0.0092, \qquad H_{\text{Q1}} = 498.3.$$

Headcount *rises* in Q1 despite the overhang, because the immediate response to rising applications outweighs the slow correction. This is what lenders historically do, and as 1.2 shows, it lands exactly in the quarter when closings are at their low.

**Central path, all four quarters:**

| Quarter | Q4 2026 | Q1 2027 | Q2 2027 | Q3 2027 |
|---|---|---|---|---|
| Workload $W_q$ | 3,310 | 3,705 | 3,998 | 3,733 |
| Headcount $H_q$ (ECM) | 493.8 | 498.3 | 502.6 | 489.8 |
| Equilibrium $H^*_q$ | 425.3 | 457.7 | 480.9 | 460.0 |
| Overhang $H_q/H^*_q - 1$ | 16.1% | 8.9% | 4.5% | 6.5% |
| Loaded cost per head $\bar c_q$ (\$) | 34,289 | 34,580 | 34,874 | 35,171 |
| Payroll $F^{\text{pers}}_q$ (\$ millions) | 16.93 | 17.23 | 17.53 | 17.23 |

Here $\bar c_q = 34{,}000 \times 1.0085^{q}$ for the $q$-th quarter ahead. Payroll barely moves over the year, while closings (1.3) fall 23% from Q3 to Q4 and 31% to Q1. That mismatch is the whole story of 1.2.

### Output

Quarterly headcount $H_q$, equilibrium headcount $H^*_q$, overhang, and payroll $F^{\text{pers}}_q = H_q \bar c_q$, on any workload path, plus the shock distribution $\zeta_q$ for 1.5.

### Hand-off

$F^{\text{pers}}_q$ goes to 1.2. $H^*_q$ (across simulated paths) goes to the report as the recommended staffing range, and the overhang and half-life go to the report as staffing overhang duration.

### Validation

Re-estimate on data through 2019 and forecast 2020–2026 headcount from actual workload: the 2020–21 refinance boom and 2022–23 bust are a severe test of $\alpha$ and $\delta$. Re-run the ADF test on rolling windows to check that the long-run relationship has not broken (for example because automation lowered staff per file, which would show up as a drifting intercept).

---

## Step 1.2: Cost per loan (cost function with partial pooling)

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Payroll | $F^{\text{pers}}_q$ | Step 1.4 |
| Non-personnel fixed cost | $K$ | Lender's P&L (technology, occupancy, compliance) |
| Closings | $C_q$ | Step 1.3 |
| Government share of closings | $s_q$ | Step 1.3 |
| Peer panel of quarterly cost per loan by lender | $\text{CPL}_{g,q}$ | MBA Quarterly Performance Report (paid; reports production cost by lender and by peer group) |
| Revenue per loan | $r$ | Lender's P&L |

**Terms.** **Fixed cost** in the short run is cost that does not change with this quarter's number of loans (salaried payroll, leases, licenses). **Variable cost** scales with each loan closed (commissions, credit reports, per-loan technology fees). **Gain on sale** is the profit from selling a closed loan to an investor. A **lender group** $g$ is a set of comparable lenders (by size tier and channel: retail, consumer-direct, wholesale), indexed so each lender belongs to group $G(g)$.

### Process

**The cost equation.** For lender (or lender group) $g$ in quarter $q$:

$$\text{CPL}_{g,q} = \frac{F^{\text{pers}}_{g,q} + K_g}{C_{g,q}} + v_g + \gamma\, s_{g,q} + e_{g,q}, \qquad e_{g,q} \sim \mathcal{N}(0, \sigma_e^2).$$

- The first term is **fixed cost spread over volume**. It is the term that explodes when $C$ falls: it is proportional to $1/C$.
- $v_g$ is the variable cost per loan.
- $\gamma$ is the extra per-loan cost of a government loan (more documentation, more conditions, longer processing), and $s_{g,q}$ the government share of closings.
- $e_{g,q}$ is everything else: one-off expenses, accruals, costs of a system migration.

**How it is estimated.** In the peer panel, payroll and other fixed costs are observed directly from cost statements, so the fixed-cost term enters as a known **offset** (its coefficient fixed at 1). What is estimated is $v_g$, $\gamma$ and $\sigma_e$. A useful diagnostic is to free the offset's coefficient: if it comes out well below 1, part of what was called "fixed" payroll actually flexes within the quarter (overtime, contractors), and $F^{\text{pers}}$ should be split.

**Partial pooling.** One lender's variable cost estimated from its own few quarters is noisy; assuming it equals its group average ignores real differences. The hierarchy

$$v_g \sim \mathcal{N}\big(v_{G(g)},\, \omega_v^2\big)$$

pulls each lender's estimate toward its group, more strongly the fewer quarters it has. Conditional on the variances, the estimate is a precision-weighted average:

$$\hat v_g = w_g\, \bar y_g + (1 - w_g)\, v_{G(g)}, \qquad w_g = \frac{n_g/\sigma_e^2}{n_g/\sigma_e^2 + 1/\omega_v^2},$$

where $\bar y_g$ is the lender's own average of $\text{CPL} - (F^{\text{pers}} + K)/C - \gamma s$ over its $n_g$ quarters. In the full model this is fit jointly by MCMC (Markov chain Monte Carlo) with weakly informative priors on $\omega_v$ and $\sigma_e$.

**Breakeven.** With revenue $r$ per loan, quarterly production profit is $(r - \text{CPL}_q)\,C_q$. Setting it to zero (ignoring $e$) and solving for volume:

$$C^*_q = \frac{F^{\text{pers}}_q + K}{r - v - \gamma s_q}.$$

The denominator is the **contribution margin** per loan: what each closing contributes toward fixed costs. Below $C^*$ the lender loses money on production.

### Worked example

**Lender L's variable cost.** L has 8 quarters of its own data, giving $\bar y_L = \$4{,}632$; its group (mid-size retail independents) has $v_G = \$4{,}420$; lenders within the group differ by $\omega_v = \$250$; quarterly noise is $\sigma_e = \$300$.

$$w_L = \frac{8/300^2}{8/300^2 + 1/250^2} = \frac{88.9 \times 10^{-6}}{88.9 \times 10^{-6} + 16.0 \times 10^{-6}} = 0.847,$$

$$\hat v_L = 0.847 \times 4{,}632 + 0.153 \times 4{,}420 = \$4{,}600, \qquad \operatorname{sd}(\hat v_L) = \sqrt{1/(104.9 \times 10^{-6})} = \$98.$$

Eight quarters of data earn 85% weight on L's own figure. The panel also gives $\gamma = \$1{,}050$.

**Q3 2026: reconciling to actual.** Payroll $540 \times 34{,}000 = \$18.36$ million, closings 3,644, government share 29.4%:

$$\text{CPL}_{\text{Q3}} = \frac{18.36\text{M} + 4.40\text{M}}{3{,}644} + 4{,}600 + 1{,}050(0.294) = 6{,}247 + 4{,}600 + 308 = \$11{,}155.$$

L's actual Q3 cost per loan was \$11,500, so $e_{\text{Q3}} = +\$345$: about one standard deviation, unremarkable.

**Q4 2026 on the central path.**

$$\text{CPL}_{\text{Q4}} = \frac{16.93\text{M} + 4.40\text{M}}{2{,}810} + 4{,}600 + 1{,}050(0.298) = 7{,}590 + 4{,}600 + 313 = \$12{,}503.$$

Personnel cost is $F^{\text{pers}}/C = \$6{,}025$ plus about \$2,700 of commission inside $v$, so about **70% of cost per loan is compensation**, consistent with the industry's two-thirds.

**Breakeven in Q4.**

$$C^*_{\text{Q4}} = \frac{21.33\text{M}}{12{,}300 - 4{,}600 - 313} = \frac{21.33\text{M}}{7{,}388} = 2{,}887 \text{ closings}.$$

The central path gives 2,810, about 77 short, for a production loss of $(12{,}300 - 12{,}503) \times 2{,}810 = -\$0.57$ million.

**Sensitivity.** The derivative of cost per loan with respect to volume is $-(F^{\text{pers}} + K)/C^2$. At Q4 levels, $21.33\text{M}/2{,}810^2 = \$2.70$ per loan for each lost closing: **every 100 fewer closings adds about \$270 to cost per loan**. Because this derivative grows as $1/C^2$, the same 100 closings cost more the lower volume already is.

**Central path, all four quarters:**

| Quarter | Q3 2026 (actual) | Q4 2026 | Q1 2027 | Q2 2027 | Q3 2027 | Year ahead |
|---|---|---|---|---|---|---|
| Closings $C_q$ | 3,644 | 2,810 | 2,503 | 2,870 | 2,890 | 11,073 |
| Fixed cost per loan (\$) | 6,247 | 7,590 | 8,642 | 7,641 | 7,484 | |
| Cost per loan (\$) | 11,500 | 12,503 | 13,562 | 12,562 | 12,406 | 12,732 |
| Breakeven $C^*_q$ | | 2,887 | 2,931 | 2,972 | 2,931 | |
| Production profit (\$ millions) | +2.91 | −0.57 | −3.16 | −0.75 | −0.31 | −4.79 |

The year-ahead cost per loan is the closings-weighted average (total cost ÷ total closings). **Q1 2027 is the trough**: closings at their low (1.3) meet headcount that has just ticked up (1.4).

### Output

For any path of payroll, closings and mix: quarterly $\text{CPL}_q$, breakeven $C^*_q$, and production profit. Also the shock distribution $e_{g,q}$ for 1.5, and the estimated $v_g$, $\gamma$ for the report.

### Hand-off

To 1.5, which evaluates this equation on every simulated path. The same equation, estimated across the peer panel at a point in time, becomes the peer curve for benchmarking in 1.5.

### Validation

Out-of-time: forecast each lender's next-quarter cost per loan from its actual payroll and closings and compare with the reported figure; the errors should have standard deviation near $\hat\sigma_e$ and no trend. Check that the offset coefficient is close to 1. Check partial pooling by holding out a lender's later quarters: the pooled $\hat v_g$ should predict them better than either the raw lender average or the group average.

---

## Step 1.5: Scenario simulation and benchmarking

### Inputs

| Input | Symbol | Source |
|---|---|---|
| Forward 10-year Treasury yields | | Federal Reserve's GSW zero-coupon curve (Gürkaynak–Sack–Wright), converted to forward rates |
| Current mortgage spread over the 10-year | | PMMS minus the 10-year Treasury yield |
| Volatility of monthly mortgage rate changes | $\sigma_X$ | Standard deviation of monthly PMMS changes, or swaption implied volatility ÷ $\sqrt{12}$ |
| Persistence of rate deviations | $\rho_X$ | From a chosen half-life |
| The chain 1.1 → 1.3 → 1.4 → 1.2 and its shock distributions | | Steps 1.1–1.4 |
| Peer cross-section of cost per loan, volume and mix | | MBA Quarterly Performance Report |

**Terms.** A **forward rate** is the interest rate for a future period implied by today's yield curve; it is the market's break-even expectation, not a forecast of what will happen. A **swaption** is an option on an interest-rate swap; its price implies the market's expected volatility of rates. A **fan chart** shows the forecast median with widening bands (P10 to P90) over the horizon.

### Process, part 1: the rate scenario simulator

The mortgage rate in month $t$ is the forward-implied level plus a random deviation:

$$m_t = f_t + X_t, \qquad X_t = \rho_X X_{t-1} + \sigma_X z_t, \qquad z_t \sim \mathcal{N}(0, 1), \qquad X_0 = 0.$$

- $f_t$ is the **central path**: the forward 10-year Treasury yield for month $t$ plus the current mortgage spread. Using the forward curve means the simulation does not bet on a direction; it is centered on what markets price.
- $X_t$ is an AR(1) deviation. With $\rho_X$ close to 1 it behaves almost like a random walk over a year (rate moves persist) but drifts back toward the forward path over several years.
- $\rho_X$ is set from a **half-life** $\tau_{1/2}$ in months: $\rho_X = 0.5^{1/\tau_{1/2}}$.
- The spread of $X_t$ after $h$ months is $\sigma_X \sqrt{\sum_{j=0}^{h-1} \rho_X^{2j}}$.

We write $\rho_X$ rather than the context's plain $\rho$ to keep it distinct from pull-through $\rho_t$.

This is a one-factor model: it moves the whole rate level and ignores changes in the spread or curve shape. That is adequate for a volume forecast. The Hull–White model used in services 4 and 5 is needed there because those services price options on rates; this one is not.

### Process, part 2: Monte Carlo through the chain

For each path $n = 1, \dots, N$ (we use $N = 20{,}000$):

1. **Rates.** Draw $z_{n,t}$ and build $m_{n,t}$ for November 2026 through October 2027 (October 2026 is fixed at 7.30%; October 2027 is needed for the September 2027 cohort's $\Delta m$).
2. **Applications (1.1).** Purchase: $\eta_{n,t} = \hat\phi\,\eta_{n,t-1} + \varepsilon_{n,t}$ starting from $\hat\eta_T$, and $P_{n,t} = \exp(\mathbf{x}_{n,t}'\hat{\boldsymbol\beta} + \eta_{n,t})$ with the path's rate inside $\mathbf{x}$. Refinance: S-curve over the buckets at $m_{n,t}$, times $e^{\nu_{n,t} - 0.10^2/2}$ (the $-\sigma^2/2$ keeps the noise mean-one), with burnout updating the buckets along the path.
3. **Closings (1.3).** Pull-through for each cohort using that path's $\Delta m_{n,t}$; closings by the lag convolution.
4. **Headcount (1.4).** Workload by quarter, ECM with a drawn $\zeta_{n,q}$, payroll.
5. **Cost (1.2).** $\text{CPL}_{n,q}$ with a drawn $e_{n,q}$, and profit $(r - \text{CPL}_{n,q})\,C_{n,q}$.

Every model's own error is drawn alongside the rate path, so the bands reflect both rate risk and model risk. Then the forecast is the empirical distribution across the 20,000 paths: quantiles of $\text{CPL}_q$ by quarter (the fan chart), and the loss probability

$$\Pr(\text{loss}) = \frac{1}{N}\sum_{n=1}^{N} \mathbf{1}\big\{(r - \text{CPL}_{n,q})\,C_{n,q} < 0\big\},$$

where $\mathbf{1}\{\cdot\}$ equals 1 when the condition holds and 0 otherwise.

### Worked example: the rate simulator

The 10-year Treasury is about 4.85% and PMMS 7.30%, a spread of 2.45 pp. The forward curve implies the 10-year easing to about 4.65% by next autumn as the oil shock is expected to fade; holding the spread constant, the central path is $f_t = 7.30 - 0.018\,(t - 1)$ for $t = 1$ (October 2026) onward:

| | Oct | Nov | Dec | Jan | Feb | Mar | Apr | May | Jun | Jul | Aug | Sep |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| $f_t$ (%) | 7.300 | 7.282 | 7.264 | 7.246 | 7.228 | 7.210 | 7.192 | 7.174 | 7.156 | 7.138 | 7.120 | 7.102 |

$\sigma_X = 0.20$ pp (the standard deviation of monthly PMMS changes over 2022–2026, and also roughly a 70 bp annual swaption volatility divided by $\sqrt{12}$). Half-life 24 months, so $\rho_X = 0.5^{1/24} = 0.9715$.

By September 2027, 11 monthly shocks have accumulated:

$$\operatorname{sd}(X) = 0.20\sqrt{\sum_{j=0}^{10} 0.9715^{2j}} = 0.20\sqrt{8.38} = 0.58 \text{ pp}.$$

So the September 2027 rate has P10 6.36%, P50 7.10%, P90 7.85% in the simulation. A rally to the low 6s, which would reopen refinancing, is roughly a one-in-ten outcome.

### Worked example: Monte Carlo results

**Fan chart, cost per loan by quarter** (20,000 paths):

| Quarter | P10 | P50 | P90 | Mean | Central path | $\Pr(\text{loss})$ |
|---|---|---|---|---|---|---|
| Q4 2026 | \$12,097 | \$12,503 | \$12,906 | \$12,502 | \$12,503 | 74% |
| Q1 2027 | \$12,781 | \$13,528 | \$14,225 | \$13,515 | \$13,562 | 98% |
| Q2 2027 | \$11,641 | \$12,510 | \$13,276 | \$12,482 | \$12,562 | 63% |
| Q3 2027 | \$11,325 | \$12,340 | \$13,175 | \$12,291 | \$12,406 | 52% |
| **Year ahead** | **\$12,097** | **\$12,683** | **\$13,160** | **\$12,650** | **\$12,732** | **81%** |

| Quarter | Closings P10 / P50 / P90 | Headcount (ECM) P10 / P50 / P90 | Equilibrium headcount $H^*$ P10 / P50 / P90 |
|---|---|---|---|
| Q4 2026 | 2,739 / 2,812 / 2,892 | 483 / 494 / 507 | 409 / 426 / 446 |
| Q1 2027 | 2,313 / 2,515 / 2,782 | 482 / 500 / 521 | 431 / 459 / 498 |
| Q2 2027 | 2,591 / 2,890 / 3,350 | 482 / 504 / 533 | 448 / 484 / 537 |
| Q3 2027 | 2,573 / 2,915 / 3,525 | 467 / 492 / 529 | 424 / 463 / 528 |

Year-ahead production profit: P10 −\$9.0 million, P50 −\$4.3 million, P90 +\$2.5 million, mean −\$3.6 million.

Things to notice:

- **The fan widens fast.** Q4 is narrow (P10 to P90 spans \$800) because its closings come mostly from applications already taken. By Q2 the band spans \$1,600, because those closings come from applications not yet taken at rates not yet known.
- **The central path is *above* the median, and the mean is lowest.** Two convexities pull in opposite directions. Cost per loan is convex in closings ($1/C$), which pushes the mean up. But refinance applications are convex in rates (the S-curve in 1.1), which makes good rate paths much better than bad paths are bad, and that pushes the mean down. Here the S-curve wins. Plugging in the central rate path, as a deterministic budget would, overstates expected cost slightly and misses the shape entirely.
- **Q1 2027 is a near-certain loss (98%).** Even favorable rate paths cannot rescue it, because its closings come from the December–January application trough.
- **The year is a loss in 81% of paths.** It is profitable mainly when rates rally: in the 10% of paths with the lowest average rate (about 6.55%), the year-ahead median cost per loan is \$11,956 and $\Pr(\text{loss})$ is only 10%. In the 10% with the highest (about 7.85%), the median is \$13,152 and a loss is essentially certain.

### Attribution: how much is rates, how much is model error?

Rerun the simulation twice, once drawing only rate shocks (model errors set to zero), once drawing only model errors (rates on the central path), and compare the variance of year-ahead cost per loan:

| Simulation | Variance (\$²) | sd (\$) | Share |
|---|---|---|---|
| Everything | 173,054 | 416 | 100% |
| Rate shocks only | 120,829 | 348 | 70% |
| Model errors only | 49,138 | 222 | 28% |
| Interaction (remainder) | | | 2% |

**About 70% of the uncertainty is rates.** That tells the lender what to watch: the cost outlook is mostly a bet on the rate path, not on the models.

### Worked example: what if L staffs to the forecast instead of following the ECM?

The ECM describes how lenders *typically* respond. The forecast's practical use is to do better. Replace the ECM headcount with a plan and rerun all 20,000 paths:

| Staffing plan (headcount Q4 / Q1 / Q2 / Q3) | Central-path profit | $\Pr(\text{annual loss})$ | Year-ahead CPL P10 / P50 / P90 |
|---|---|---|---|
| Historical behavior (ECM): 494 / 498 / 503 / 490 | −\$4.79M | 81% | \$12,097 / \$12,683 / \$13,160 |
| Plan A: 470 / 455 / 455 / 455 | +\$0.41M | 42% | \$11,468 / \$12,205 / \$12,794 |
| Plan B: 460 / 445 / 450 / 445 | +\$1.63M | 34% | \$11,370 / \$12,096 / \$12,678 |

Reaching Plan A means about 70 fewer salaried staff by the end of Q4 than today's 540, which at typical industry turnover of 3–4% a quarter is mostly attrition plus a hiring freeze, with some reductions. Two caveats belong in the report: severance and other separation costs are not in this calculation, and a fixed plan assumes L can still process volume in a rally. In the P90 rate path, equilibrium headcount reaches 537 in Q2, so a flexible bench (contract underwriters, cross-trained closers) is the hedge for the upside.

### Process, part 3: benchmarking against peers

The client also wants to know *where it stands*: is its cost high because it is small (a **scale** problem it can only fix by growing) or because it runs inefficiently for its size (an **efficiency** problem it can fix internally)?

**Peer percentile.** Rank the client's cost per loan among peers in the same quarter.

**Peer curve.** Fit the cost equation across peers at one point in time:

$$\widehat{\text{CPL}}(C, s) = \frac{A^{\text{peer}}}{C} + B^{\text{peer}} + \gamma\, s,$$

by least squares on regressors $1/C$, a constant, and $s$. $A^{\text{peer}}$ is not any one lender's fixed cost (larger lenders have larger payrolls); it measures how strongly cost per loan falls with scale *across* lenders. $B^{\text{peer}}$ is the per-loan cost a very large lender would approach.

**Decomposition.** With $\text{CPL}_{\text{median}}$ the peer median, the client's gap splits exactly into two parts:

$$\text{CPL}_{\text{client}} - \text{CPL}_{\text{median}} = \underbrace{\widehat{\text{CPL}}(C_{\text{client}}) - \text{CPL}_{\text{median}}}_{\text{scale effect}} + \underbrace{\text{CPL}_{\text{client}} - \widehat{\text{CPL}}(C_{\text{client}})}_{\text{efficiency gap}}.$$

The scale effect is what a typical lender of the client's size and mix would cost relative to the median. The efficiency gap is how far the client sits above or below the curve for its own size.

### Worked example: benchmarking Lender L (Q3 2026)

Peer cross-section: 120 lenders, median volume 3,080 closings a quarter, median cost per loan **\$11,995** (in line with the industry's roughly \$11,900). Least-squares fit:

$$\widehat{\text{CPL}}(C, s) = \frac{15{,}102{,}000}{C} + 6{,}501 + 1{,}039\, s, \qquad R^2 = 0.97.$$

The cross-sectional $\gamma = \$1{,}039$ (s.e. \$406) agrees with the panel's \$1,050 in 1.2.

At L's Q3 volume and mix:

$$\widehat{\text{CPL}}(3{,}644,\ 0.294) = \frac{15{,}102{,}000}{3{,}644} + 6{,}501 + 1{,}039(0.294) = 4{,}145 + 6{,}501 + 305 = \$10{,}951.$$

| Component | Amount (\$ per loan) |
|---|---|
| L's actual cost per loan | 11,500 |
| Peer median | 11,995 |
| Gap to median | −495 |
| Scale effect: $10{,}951 - 11{,}995$ | **−1,044** |
| Efficiency gap: $11{,}500 - 10{,}951$ | **+549** |

**Peer percentile:** 44% of peers had lower cost per loan, so on the headline number L looks better than average. The decomposition says otherwise: L's size *should* buy it a cost about \$1,044 below the median, and it gives back more than half of that through inefficiency. Its efficiency gap is larger than 87% of peers' (measured as each peer's distance from the curve). Part of that gap is the staffing overhang from 1.4: L's 25 people above equilibrium in Q3 cost $25 \times 34{,}000 / 3{,}644 = \$233$ per loan.

### Output

Quarterly fan chart of cost per loan; $\Pr(\text{loss})$ by quarter and for the year; breakeven volumes; distribution of equilibrium headcount (the recommended staffing range); the rate vs. model-error attribution; results under alternative staffing plans; peer percentile with the scale/efficiency split.

### Hand-off

To the report. The rate simulator is a **shared component**: the same one-factor model supplies rate scenarios to 1.1 here and to the affordability stress in 2.5 (Escrow).

### Validation

Backtest the full chain: run it as of each quarter-end from 2018 to 2025 using only data available then, and check that about 80% of realized quarterly cost per loan falls in the P10 to P90 band, and that quarters given $\Pr(\text{loss}) \approx 0.7$ were losses about 70% of the time. Validate each upstream step on its own first (1.1, 1.3, 1.4, 1.2), since errors compound through the chain. For the rate simulator, check that the simulated distribution of 12-month rate changes matches history in spread (coverage of realized changes) without assuming it matches in direction.

---

## The report

### Lender card (Lender L)

- **Cost per loan:** \$11,500 in Q3 2026. Forecast \$12,503 in Q4 2026 and \$13,528 in Q1 2027 (P10 \$12,781, P90 \$14,225), easing to about \$12,300–12,500 by mid-2027. Year-ahead \$12,683 (P10 \$12,097, P90 \$13,160).
- **Why:** closings fall 23% in Q4 and 31% by Q1 from Q3's level as the September rate spike ends the summer refinance wave and the seasonal trough arrives, while payroll stays nearly flat. Every 100 lost closings adds about \$270 to cost per loan at current volume.
- **Breakeven:** about 2,890 closings in Q4 and 2,930 in Q1; the forecast is 2,810 and 2,500.
- **Profit risk:** 81% chance of a production loss over the next four quarters if staffing follows historical behavior; Q1 2027 is a loss in 98% of scenarios.
- **What drives the uncertainty:** about 70% interest rates, 28% model error. The upside case is a rally into the low 6s, which reopens refinancing for the 7%-plus borrowers.
- **Staffing:** equilibrium headcount for the forecast workload is 426 in Q4 (P10–P90 409–446) and 459 in Q1 (431–498), against 540 today. Recommended path: about 470 by end of Q4 and 455 from Q1, mostly via attrition and a hiring freeze, with a flexible bench for a rally. That plan cuts the loss probability from 81% to about 42%.
- **Staffing overhang duration:** at historical adjustment speed, the overhang is 16% after Q4 and halves only every 4.3 quarters; left alone, L stays overstaffed through 2027.
- **Benchmark:** 44th percentile on cost per loan (cheaper than 56% of peers), but its scale should place it about \$1,044 below the peer median and inefficiency gives back \$549 of that. About \$233 of the gap is the current staffing overhang.

### Fallout cost by segment

Labor spent on applications that never close is paid for by the loans that do. At Q4 2026 payroll and workload, one workload unit costs $F^{\text{pers}}/W = 16.93\text{M}/3{,}310 = \$5{,}116$ of salaried labor, so a purchase application costs about \$5,116 and a refinance application about \$3,581 ($\omega_R = 0.7$). Spreading the cost of failed applications over the successful ones in the same segment gives a fallout cost per closed loan of $\text{labor per application} \times (1 - \rho_s)/\rho_s$:

| Segment | Pull-through $\rho_s$ | Labor per application | Fallout labor per closed loan |
|---|---|---|---|
| Conventional purchase | 74.3% | \$5,116 | \$1,772 |
| Government purchase | 67.1% | \$5,116 | \$2,506 |
| Conventional refinance | 59.5% | \$3,581 | \$2,440 |
| Government refinance | 50.9% | \$3,581 | \$3,453 |

This is an upper bound, since an application that drops out early uses less than a full file's labor; a lender with stage-level pipeline data can replace it with actual hours. Even so, the ranking is the actionable part: government refinances carry nearly twice the fallout burden of conventional purchases, which argues for earlier screening (pre-underwriting DTI and documentation) in that segment.

### Portfolio and investor view

For an investor in lender stocks or a technology vendor, the same engine run over a panel of lenders gives a table like the lender card for each: year-ahead cost per loan fan, $\Pr(\text{loss})$, overhang, and the scale/efficiency split. Lenders with a large efficiency gap and a large overhang are the ones most likely to announce layoffs or seek a merger; lenders whose profitability depends on the P90 rate path are effectively making a rate bet.

---

## Symbol reference

| Symbol | Meaning | Step |
|---|---|---|
| $t$, $q$, $h$, $T$ | Month, quarter, forecast horizon in months, forecast origin (September 2026) | all |
| $m_t$ | 30-year mortgage rate (PMMS), percent | 1.1, 1.3, 1.5 |
| $P_t$, $R_t$, $A_t$ | Purchase, refinance and total applications | 1.1 |
| $y_t$, $\mathbf{x}_t$, $\boldsymbol\beta$, $\beta_m$ | Log purchase applications; month-level drivers; coefficients; rate semi-elasticity | 1.1 |
| $\eta_t$, $\phi$, $\varepsilon_t$, $\sigma_\varepsilon$ | AR(1) regression error; its persistence; innovation; innovation sd | 1.1 |
| $k$, $c_k$, $w_{k,t}$, $N_t$ | Note-rate bucket; its note rate; its share; catchment size | 1.1 |
| $I_{k,t}$ | Refinance incentive $c_k - m_t$ | 1.1 |
| $\pi(I)$, $\pi_0$, $\pi_{\max}$, $I_0$, $\kappa$ | S-curve; cash-out floor; maximum; midpoint; scale | 1.1 |
| $\nu_t$ | Refinance model noise | 1.1, 1.5 |
| $i$, $\mathbf{a}_i$, $\boldsymbol\theta$, $p_i$ | Application; its features; logistic coefficients; its origination probability | 1.3 |
| $\Delta m_i$, $\Delta m_t$ | Rate change in the month after application | 1.3 |
| $\rho_s$, $\rho_{\text{purch}}$, $\rho_{\text{refi}}$, $\rho_t$ | Segment pull-through; purchase; refinance; monthly blend | 1.3 |
| $\ell$, $\lambda_\ell$ | Months from application to closing; share closing at lag $\ell$ | 1.3 |
| $C_t$, $C_q$ | Closings by month, by quarter | 1.3, 1.2 |
| $s_q$ | Government (FHA/VA) share of closings | 1.3, 1.2 |
| $W_q$, $\omega_P$, $\omega_R$ | Workload; labor weights of purchase and refinance applications | 1.4 |
| $H_q$, $H^*_q$ | Salaried headcount; equilibrium headcount | 1.4 |
| $a_H$, $a_L$, $b$ | Long-run intercept (industry, lender); long-run elasticity | 1.4 |
| $u_q$, $\alpha$, $\delta$, $\zeta_q$, $\sigma_H$ | Disequilibrium; adjustment speed; immediate response; shock; shock sd | 1.4 |
| $\bar c_q$, $F^{\text{pers}}_q$ | Loaded cost per employee; payroll | 1.4, 1.2 |
| $g$, $G(g)$ | Lender (or lender group); its peer group | 1.2 |
| $\text{CPL}_{g,q}$ | Production cost per loan | 1.2 |
| $K$ | Non-personnel fixed cost per quarter | 1.2 |
| $v_g$, $v_{G}$, $\omega_v$, $w_g$, $\bar y_g$ | Variable cost per loan; group mean; spread across lenders; pooling weight; lender's own average | 1.2 |
| $\gamma$ | Extra cost of a government loan | 1.2, 1.5 |
| $e_{g,q}$, $\sigma_e$ | Cost equation error; its sd | 1.2 |
| $r$, $C^*_q$ | Revenue per loan; breakeven closings | 1.2, 1.5 |
| $f_t$, $X_t$, $\rho_X$, $\sigma_X$, $z_t$ | Forward-implied rate path; deviation; its persistence; its monthly sd; standard normal shock | 1.5 |
| $N$, $n$ | Number of simulated paths; path index | 1.5 |
| $A^{\text{peer}}$, $B^{\text{peer}}$, $\widehat{\text{CPL}}(C, s)$ | Peer curve scale coefficient; per-loan floor; fitted peer cost | 1.5 |
