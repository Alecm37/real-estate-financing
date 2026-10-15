
# **Step 1 - Volume forecasting 
### Overview: 
The goal of this step is to forecast monthly purchase and refinance applications given a specific rate context, since the two factors have different drivers. 
### Inputs:
- Historical Monthly Purchase and refinance applications
- Existing home sales
- 30-year mortgage rates
- Distribution of outstanding mortgage rates per rate bucket
- Future rate path (step 5)
### Purchase and refinancing applications, regression with autocorrelated errors
**Regression:**
Consider that we are attempting to predict the log number of purchase applications in a month $y_t$:
$$y_y=\beta_0+\beta_1\log(\text{HomeSales}_t)+\beta_2m_t+\sum_{j=1}^{11}\gamma_jD_j+\eta_t$$
Where $m_t$ is the mortgage rate and $D_{j,t}$ are a calendar-month indicator weighting since spring is always the hottest month. Each $\beta$ corresponds to a driver: store it in a vector $x_t$ for convenience.
**Error Part:**
The standard regression assumes leftover error $\eta_n$ is independent across months, which is not the cause: errors in this case are usually correlated with each other. So, model the error as an autoregressive process:
$$\eta_t=\phi\eta_{t-1}+\epsilon_t, ~~~~0\leq\phi<1$$
Here $\phi$ measures how much of last month's error is attributed to this month's error, and $\epsilon_t$ is our actual definitive, isolated shock. In order to make estimated predictions, we have to estimate them jointly:
$$y_t-\phi y_{t-1}=(x_t-\phi x_{t-1})'\beta+(\eta_t-\phi\eta_{t-1})$$
Now $\epsilon_t=\eta_t-\phi\eta_{t-1}$. We an solve for $\beta$ and $\phi$ simultaneously by using maximum likelihood estimation. This involves finding the maximum of the log-distribution in terms of the two variables. 

**Forecasting:**  The number of purchases $h$ months past the last observation at $T$ is provided by:
$$\hat{y}_{T+h}=x'_{T+h}\hat{\beta}+\hat{\phi}^h\hat{\eta}_T$$
The first term uses the future driver values of the regression and the rate path (talked about in step 4), and the second term carries the latest shock forward, shrinking by a factor of $\phi$ every month. 

**Ex:** The last month applications came in 10 percent above the model ($\hat\eta_T=0.10$) and $\hat\phi=0.6$: hot streak fades over time gradually, by $0.6$ a month. So this is how we forecast the number of purchase applications, but what about the number of refinance applications?

**Refinance Applications:** Refinance demand depends on how borrowers are able to save their money, which in and of itself depends on the distribution of existing mortgage rates. We build this bottom-up:
An incentive, for a rate bucket $k$ with a corresponding rate $c_k$:
$$I_{k,t}=c_k-m_t$$
The S-curve gives the monthly probability that a borrower in bucket $k$ applies to refinance:
$$\pi(I)=\frac{\pi_{max}}{1+e^{-(I-I_0)/s}}$$
- $\pi_{max}$ is the maximum probability
- $I_0$ is the midpoint: the incentive where the probability is half the ceiling
- $s$ controls the steepness of the curve
The total number of refinance applications with $N_t$ outstanding loans and $w_{k,t}$, the share in bucket $k$:
$$R_t=N_t\sum_kw_{k,t}\pi(I_{k,t})$$
Use historical reference values to estimate $\pi_{max}$, $I_0$, and $s$ by minimizing the squared error between predicted and actual $R_t$ (standard OLS regression). 
# Step 2: Pull-through: applications to closed
### Overview:
The goal of this is to measure how much processed work never actually earns revenue in the process of closing applications.
### Inputs:
- HMDA application level-records
- Predicted number of purchase and refinance applications from the previous step
### Logistic Regression/Gradient Boosting
**Goal:** For application $i$, $z_i=1$ if originated and $0$ otherwise. 
**Model:** Compute a linear score from features $x_i$, then map it to a probability $p_i$ which is the probability that application $i$ closes:
$$p_i=\text{Pr}(z_i=1|x_i)=\frac{1}{1+e^{-x'_i\theta}}$$
The feature vector $x_i$ represent a single application $i$ from the HMDA, with characteristics:
$$x_i=(1,\text{Refi}_t, \text{FHA}_i,\text{VA}_i, \text{DTI}_i, \text{LTV}_i,\log(\text{LoanAmount}_i), \text{LenderType}_i,\text{Year}_i)$$
We then use MLE to choose the value of $\theta$ which maximizes 
$$\sum_i\Big[z_i\log p_i+(1-z_i)\log(1-p_i)\Big]$$
Then calculate the segment rate of all purchase applications that close and refinance applications that close, which uses this above formula:
$$\rho_{\text{purch}}=\frac{1}{n_{\text{purch}}}\sum_{i\in\text{purchase apps}}p_i$$
$$\rho_{\text{refi}}=\frac{1}{n_{\text{refi}}}\sum_{i\in\text{refinance apps}}p_i$$

Then we calculate the monthly blended probability of applications that close for month $t$:
$$\rho_t=\frac{P_t}{A_t}\rho_{\text{purch}}+\frac{R_t}{A_t}\rho_{\text{refi}}$$
Where $A_t=R_t+P_t$. The total number of closed applications is then given by:
$$C_t=\sum_{k=0}^K\lambda_k\rho_{t-k}A_{t-k},~~~~\sum_{k=0}^K\lambda_k=1$$
We calculate a summation from every $k$  using specified weights $\lambda_k$. So, each $\lambda$ is effectively the probability that a loan takes $k$ months to close: we are solving for the probability that a loan closes given that it originated $k$ months ago. 

How are these weights estimated? HMDA records the application date and the action date for each application. For only originated loans:
- Compute the lag in calendar months (action date - application date)
- Count how many loans have each lag value, then divide by the total
$$\lambda_k=\frac{\text{number originated loans with }L_i=k}{\text{number originated loans}}$$
# Step 3: Staffing costs
### Overview: 
Forecast production headcount, understanding that lenders hire and fire slowly.
### Inputs:
- Historical quarterly headcount
- Historical quarterly workload
- Forecast workload path
### Process:
First calculate the forecasted workload path: 

