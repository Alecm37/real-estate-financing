# Labor Cost Forecasting
**Possible Clients:** Loan lenders planning project staffing, investors evaluating lender stocks, technology vendors (resellers)
**Main Idea:** Cost per loan is a static price over volume problem: cost doesn't really fluctuate over the volume of a loan, so when loan volume plummets, the cost per loan spikes. The solution for this is to forecast volume and then estimate how the loan cost responds.
**Step 1 - Volume forecasting (regression with time-series errors):**
**Step 2 - Cost function estimation (hierarchical regression):**
**Step 3** - **Probability an application for a loan actually closes (logistic regression or gradient boosting)**
**Step 4 - Staff-adjustment lag (error correction model)
**Step 5 - Scenario simulation and benchmarking (Monte Carlo)**
**Report:** Expected cost per loan with uncertainty bands under rate scenarios, breakeven volume, recommended staffing range and timing, cost attributable to fallout and overcapacity 
# Escrow Forecasting
**Possible Clients:** Mortgage services that run escrow accounts, lenders checking qualifications, credit investors checking delinquency risk, and homebuyers or agents. 
**Main Idea:** Escrow is property tax plus insurance, but the two behave completely differently: tax is the application of legal rules to market value, whereas insurance follows actuarial pricing plus occasional jumps. They must be modeled separately and then combined. 
**Step 1 - Tax projection (rules engine plus valuation model)**
**Step 2 - Tax rate forecasting (hierarchical Bayesian time series)** 
**Step 3 - Insurance premium modeling (general linear models and a filing-lag model)**
**Step 4 - Premium jump risk (classification plus a mixture model)** 
**Stage 5- Combination and escrow mechanics (Monte Carlo and quantile outputs)** 
**Report:** Expected change in escrow reports with uncertainty bands, breakdown of how much risk comes from insurance vs. tax, and which conditions push borrowers over their affordability limits, suggesting delinquencies. 
# Climate Pricing and Price Discovery 
**Possible Clients:** Buyers and investors, lenders and insurers, single-family rental and homebuilder funds, and regional banks. 
**Main Idea:** Partition a house's value into its intrinsic value (ignoring climate risk), full cost of its climate risk, how much of the cost is already priced into the market, revealing mis-pricing. 
**Step 1 - Automated valuation (gradient boosting + repeat sales + spatial smoothing)**
**Step 2 - Risk quantification (expected loss from damage functions)** 
**Step 3 - Capitalization (causal inference)**
**Step 4 - Mis-pricing and adjustment speed (partial adjustment model)** 
**Step 5 - Stress testing and portfolio aggregation (Monte Carlo)**
**Report:** Fair value with a confidence range, climate-adjusted value, how much risk is priced in vs. not, adjustment timeline, stress-scenario losses, where risk is concentrated inside portfolios. 
# Loan Transfers (Assumable Mortgages)
**Possible Clients:** Sellers and listing agents, buyers, lenders and servicers, and investors. 
**Main Idea:** An assumable low-rate loan is attached to a property. The model finds these loans, prices them, and then assesses if the loan can actually close or not. 
**Step 1 - Identification (probabilistic record linkage and text classification)**
**Step 2 - Balance reconstruction (amortization math)**
**Step 3 - Valuation (survival mode and rate simulation)**
**Step 4 - Market pricing test (matching estimators)**
**Step 5 - Deal feasibility (financing optimization plus a timeline model)**
**Report:** The loan's realistic value and how much to market it for, the loan rate and whether the premium is justifiable or not, a map of geographic loan concentration and consequent gaps
# MBS Turnover Forecasting 
**Possible Clients:** MBS investors and mortgage REITs, servicers, GSEs, and brokerages aggregated by zip code
**Main Idea:** Loans pay off for different reasons and because of different drivers. Isolate specific pushovers and hazards, allocated signals into pools, and then differentiate. 
**Step 1 - Payoff decomposition (latent-class labeling)** 
**Step 2 - Competing risk-hazard model (discrete-time multinomial logit + gradient boosting)**
**Step 3 - Local nowcasting (Dynamic factor with a kalman filter)**
**Step 4 - Aggregation and calibration (hazard to CPR, and Bayesian shrinkage)**
**Step 5 - Valuation and translation (cash-flow engine with option-adjusted speed)**
**Report:** Projected Conditional Prepayment Rates (CPR) by classification pool, how much speed comes from actual turnover vs. refinancing, which regions and loan types drive turnover surprises, a rich/cheap ranking of pools, rate sensitivity to other scenarios. 



 