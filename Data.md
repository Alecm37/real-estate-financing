### Labor Cost Forecasting
**What's being predicted:** Origination costs per loan/labor hours put into the underlying property.

| Data                                                               | Use                                                   | Source                                          | Access             |
| ------------------------------------------------------------------ | ----------------------------------------------------- | ----------------------------------------------- | ------------------ |
| Cost per loan, broken out by compensation, occupancy, technology   | The model's target values                             | MBA quarterly performance report                | Paid (MBA)         |
| Loan volume per lender                                             | Fixed costs spread over fewer loans when volume drops | HMDA, covering every US lender                  | Free               |
| Purchase vs. refinance mix                                         | Purchase loans are more labor intensive               | HMDA; MBA weekly application survey             | Free/slightly paid |
| Pull-through rate (applications that close)                        | Failed applications that cost labor with no revenue   | HMDA, recording denials and withdrawals         | Free               |
| Loan officer headcount and licensing                               | Labor supply                                          | NMLS consumer access, NMLS industry reports     | Free               |
| Wages for loan officers, processors, underwriters                  | The labor cost driver                                 | BLS occupational employment and wage statistics | Free               |
| Loan complexity (FHA/VA vs. conventional, self-employed borrowers) | Complex files inherently take longer                  | HMDA loan-type/income fields                    | Free               |
| Technology adoption (e-closing, automated underwriting)            | The cost-reducing factor                              | Industry surveys (STRATMOR, MBA)                | Paid               |
| Rates and the yield curve                                          | Drive volume swings                                   | FRED, Freddie Mac, weekly rate survey           | Free               |
**Caveat:** Cost data is self-reported, thus data represents industry averages and not necessarily specifics. Determine what data he client needs to provide.
### Escrow Forecasting 
**What's being predicted:** Next year's escrow payment/shortfall. 

| Data                                           | Why it matters                                                                                                                                                      | Source                                                                           | Access                                 |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- | -------------------------------------- |
| Current assessed value and tax bill per parcel | Baseline                                                                                                                                                            | County assessor and treasurer records                                            | Free but fragmented across counties    |
| Reassessment schedule and rules                | Some counties reassess annually, others every 4–6 years. Some reset value on sale; some cap annual increases (e.g., Florida’s Save Our Homes, California’s Prop 13) | State statutes, county assessor sites                                            | Free, requires manual coding per state |
| Millage (tax) rates and recent changes         | Rate changes hit all properties in a jurisdiction                                                                                                                   | County and municipal budgets                                                     | Free                                   |
| Market value trends                            | Assessments eventually follow market prices                                                                                                                         | FHFA House Price Index (down to zip level); Case-Shiller; Zillow home value data | Free                                   |
| Homeowners insurance rate filings              | Insurers must file rate changes with state regulators, so increases are visible _before_ they hit escrow                                                            | State insurance departments; SERFF filing access                                 | Free but messy                         |
| Insurer market exits and nonrenewals           | A forced switch often means a big premium jump                                                                                                                      | State insurance department bulletins and news                                    | Free                                   |
| Reinsurance pricing                            | Reinsurance costs flow into consumer premiums with a lag                                                                                                            | Guy Carpenter rate-on-line index, industry reports                               | Mostly paid; summaries free            |
| Catastrophe exposure                           | Drives premiums                                                                                                                                                     | FEMA National Risk Index; NOAA storm data                                        | Free                                   |
| Flood insurance policies and claims            | Flood is a separate policy and cost line                                                                                                                            | OpenFEMA NFIP datasets                                                           | Free                                   |
| Actual escrow histories                        | Ground truth for training                                                                                                                                           | Servicer data (Cotality, ICE)                                                    | Paid                                   |
**Caveat:** Insurance premiums aren't public per property. You'll have to find generalized data per location in order to train the models, and then once clientele is accrued, ask them if they would be willing to provide that data.
### Climate Pricing and Price Discovery:
**What's being predicted:** A property's fair value and how much climate risk is reflected in its price. 
**Price discovery core:** 

| Data                                                   | Why it matters                          | Source                                           | Access                                  |
| ------------------------------------------------------ | --------------------------------------- | ------------------------------------------------ | --------------------------------------- |
| Transaction prices (sale price, date, parcel)          | The target variable                     | County deed records; ATTOM, Cotality, Regrid     | Free but fragmented / paid              |
| Property characteristics (size, beds, year built, lot) | Hedonic features                        | County assessor; listing data                    | Free / paid                             |
| Listing history (list price, cuts, days on market)     | Leading indicator, captures negotiation | Redfin and Realtor.com aggregate data; MLS feeds | Free aggregates; MLS requires a license |
| Neighborhood features (schools, crime, amenities)      | Location premium                        | NCES, local data portals, OpenStreetMap          | Free                                    |
| Repeat-sales indices                                   | Market-level trend                      | FHFA HPI, Case-Shiller                           | Free                                    |

**Climate layer:**

|Data|Why it matters|Source|Access|
|---|---|---|---|
|Flood zones|Legal flood-insurance requirement and risk proxy|FEMA flood maps (NFHL)|Free|
|Forward-looking flood, fire, heat, wind risk scores|FEMA maps are backward-looking and often outdated|First Street; FEMA National Risk Index|Partly free / paid|
|Wildfire hazard|Western US risk|USFS Wildfire Risk to Communities|Free|
|Historical disaster events and claims|Realized losses|OpenFEMA, NOAA Storm Events|Free|
|Elevation and sea-level rise projections|Coastal risk|USGS 3DEP elevation data; NOAA sea-level rise viewer|Free|
|Insurance cost by area|The channel through which risk hits prices (see escrow section)|Rate filings, Newrez/Cotality studies|Free summaries|
**Caveat:** Run price regressions on on changes in insurance costs and risk scores, having specifications for local trends. You need to panel data over multiple years. 
### Loan Transfers:
**What's being predicted:** Which homes have assumable loans, the worth of the loan, how quickly it's assumed to close.

|Data|Why it matters|Source|Access|
|---|---|---|---|
|Loan type (FHA, VA, USDA vs. conventional)|Only government loans are generally assumable|County recorder mortgage documents; HMDA (aggregate, not linked to address)|Free but fragmented|
|Original rate, balance, origination date, term|Needed to compute remaining balance and payment|Ginnie Mae loan-level disclosure files (for pooled FHA/VA loans); recorded mortgage amounts|Free|
|Matching loans to specific properties|Ginnie data doesn’t include addresses|Recorder data joined on lender, amount, and date; or commercial data (ATTOM, Cotality)|Hard / paid|
|Current market rates|Determines the value of the rate gap|Freddie Mac weekly rate survey|Free|
|Turnover hazard (how long the buyer keeps the loan)|Converts the naive present value into a realistic one (as in the $98,600 example)|Fannie/Freddie loan-level performance data, used to estimate payoff curves|Free|
|Equity gap (price minus assumable balance)|The buyer must cover it with cash or a second loan, which limits the buyer pool|Price estimates minus amortized balance|Derived|
|Assumption processing times and approval rates|The friction cost|Servicer disclosures, FHA/VA guidance, industry reports|Sparse|
|Assumption volume|Market size|Ginnie Mae reports; assumption-platform statistics|Partial|
|Listings that mention assumable loans|Whether the market prices it|Listing text (MLS or scraped portals, within terms of service)|License / restricted|
**Caveat:** Compare sale prices of homes with low-rate loans vs. the sale price of homes without them, revealing how effective the market prices an embedded loan. 
### MBS Turnover Forecasting
Turnover component (sales) of an MBS (macroeconomic factors). 

| Data                                           | Why it matters                                     | Source                                                                                                                                     | Access                    |
| ---------------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------- |
| Loan-level origination and monthly performance | Training data: you observe when each loan pays off | Fannie Mae Single-Family Loan Performance Data; Freddie Mac Single-Family Loan-Level Dataset                                               | Free with registration    |
| Ginnie Mae loan-level data                     | FHA/VA loans behave differently                    | Ginnie Mae disclosure files                                                                                                                | Free                      |
| Pool-level monthly prepayment factors          | What the market actually trades                    | Fannie, Freddie, and Ginnie monthly disclosures; vendors like eMBS and Recursion                                                           | Free (raw) / paid (clean) |
| Payoff reason (refinance vs. sale)             | Separates turnover from refinancing                | Not directly reported. Infer from rate incentive (out-of-the-money loans prepay mostly via sales) or by matching payoffs to recorded sales | Derived / paid            |

**Macro and local features (turnover drivers):**

| Data                                    | Why it matters                 | Source                                          | Access  |
| --------------------------------------- | ------------------------------ | ----------------------------------------------- | ------- |
| Local home price changes                | Equity enables moving          | FHFA HPI by state, metro, or zip                | Free    |
| Local employment and job changes        | Job moves trigger sales        | BLS QCEW, Census LEHD job-flow data             | Free    |
| Migration flows                         | Out-migration raises turnover  | IRS county-to-county migration data; Census ACS | Free    |
| Demographics (age, household formation) | Life-stage moves               | Census ACS                                      | Free    |
| Listing and pending-sale activity       | Near-real-time turnover signal | Redfin and Realtor.com aggregate data           | Free    |
| Seasonality                             | Summer moves dominate          | Derived from the loan data                      | Derived |
| Rate-lock distribution by area          | Share of locked-in owners      | FHFA National Mortgage Database aggregates      | Free    |
**Caveat:** More of a survival analysis process: each month; a loan has a probability of paying off, which can be modeled using the above features, and trained off of actual benchmarked rates. 

