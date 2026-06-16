# Appendix — An Agent-Based Model of Recidivism
## SBP-BRiMS 2026 Supplementary Material

Full equations, coefficient tables, and model specification details
referenced in the main paper. Omitted from the proceedings due to the
10-page LNCS limit.

---

## Appendix A: NIJ/PCRA Risk Score Coefficients

Table A1 lists the full set of NIJ-derived logistic regression
coefficients used in `compute_risk_score()` (`scoring.py`). Coefficients
mirror the PCRA instrument weights validated against the NIJ Recidivism
Forecasting Challenge dataset (NIJ, 2021; PCRA, 2016). Count variables
use log-compression $f(x) = \log(1 + x)$; binary and continuous
variables use the identity transformation. Race and gender are absent
by design (Skeem & Lowenkamp, 2016).

**Table A1. Full NIJ/PCRA logistic regression coefficients.**
Positive values increase predicted rearrest probability; negative values
are protective. †Violent collapses PCRA Violent-SexOffender
($\hat{\beta} = -0.205$) and Violent/Non-Sex ($\hat{\beta} = +0.099$)
using a 13%/87% prevalence-weighted average from BJS state-prison cohort
data.

| Category | Attribute | $\hat{\beta}$ |
|---|---|---|
| **Demographics** | Age at release (log-compressed) | $+0.730$ |
| | Percent days employed | $-0.950$ |
| | Number of dependents | $+0.161$ |
| **Education** | Less than HS diploma | $+0.154$ |
| | HS diploma | $+0.000$ |
| | At least some college | $-0.080$ |
| | No formal education | $-0.049$ |
| **Social** | Gang affiliated (binary) | $+0.840$ |
| | Residence changes (count) | $+0.211$ |
| **Offense** | Violent (collapsed†) | $+0.060$ |
| | Drug | $-0.047$ |
| | Property | $+0.137$ |
| | Other/PublicOrder | $+0.089$ |
| **Supervision** | Supervision risk score (1–10) | $+0.037$ |
| | Level: Standard | $-0.109$ |
| | Level: High | $+0.075$ |
| | Level: Specialized | $+0.108$ |
| **Prior convictions** | Violent episodes (log-compr.) | $+0.086$ |
| | Property episodes (log-compr.) | $+0.027$ |
| | Drug episodes (log-compr.) | $+0.026$ |
| | Misdemeanor episodes (log-compr.) | $+0.390$ |
| | Felony episodes (log-compr.) | $+0.242$ |
| **Prior revocations** | Prior revocations (log-compr.) | $+0.434$ |
| **Behavioral** | Mental health/substance abuse | $+0.350$ |
| | Cognitive/educational need | $-0.015$ |
| | Other condition | $+0.103$ |
| **Program & violations** | Technical violations (binary) | $+0.210$ |
| | Program attendances (log-compr.) | $-0.380$ |
| | Unexcused absences (log-compr.) | $+0.110$ |

---

## Appendix B: Model Specification

### B.1 Risk Score Construction

Each agent receives a composite risk score computed as a weighted sum of
29 criminogenic, behavioral, demographic, and supervision-related
attributes:

$$
s_i = \sum_k \beta_k \, f_k(x_{i,k})
$$

where $\beta_k$ denotes the coefficients estimated from the NIJ
Recidivism Forecasting Challenge dataset (2021) using a logistic
regression model, and $f_k(\cdot)$ is an attribute-specific
transformation function. Count-valued variables (e.g., prior convictions
and program participation counts) are log-compressed to reflect diminishing marginal risk contributions.
Binary and bounded continuous variables are used without transformation.

$$
f(x) = \log(1 + x)
$$

Race and gender are excluded from the risk score consistent with the
race-neutral design of the Post Conviction Risk Assessment (PCRA) (PCRA, 2016; Skeem & Lowenkamp, 2016).
Full attribute weights are reported in Table A1 above.

---

### B.2 Risk Tier Assignment

Risk scores are sigmoid-normalized to the interval $[0, 1]$ and mapped
to four PCRA risk tiers using population-derived quantile thresholds:

- Low
- Low-Moderate
- Moderate
- High

Each tier is associated with a fixed log-odds contrast derived from the
PCRA-to-BJS transfer procedure:

$$
c_{\text{Low}}     = -1.085 \qquad
c_{\text{LowMod}}  = +0.012 \qquad
c_{\text{Mod}}     = +0.763 \qquad
c_{\text{High}}    = +1.381
$$

The Stage 2 calibration parameter $\gamma$ controls the separation
between risk tiers. When $\gamma = 0$, all tiers experience identical
rearrest hazards despite differing risk scores. Calibration identifies
$\gamma = 1.0$, resolving this equifinality condition.

---

### B.3 Quarterly Hazard Schedule

Rearrest is evaluated quarterly using the log-odds hazard model. The
baseline quarterly hazard follows a four-block desistance schedule
derived from BJS longitudinal recidivism statistics (Alper et al., 2018):

$$
q_1 = 0.134 \quad \text{(Year 1)}
$$
$$
q_2 = 0.070 \quad \text{(Years 1–3)}
$$
$$
q_3 = 0.035 \quad \text{(Years 3–6)}
$$
$$
q_4 = 0.018 \quad \text{(Years 6–9)}
$$

The first-year hazard is obtained from the BJS one-year cumulative
rearrest rate (43.9%), and subsequent hazards are scaled using the
Stage 1 desistance parameters calibrated to reproduce the observed
3-, 6-, and 9-year cumulative rearrest rates.

---

## References

- Alper, M., Durose, M. R., & Markman, J. (2018). *2018 Update on
  Prisoner Recidivism: A 9-Year Follow-Up Period (2005–2014)*.
  BJS NCJ 250975.
- NIJ. (2021). *NIJ Recidivism Forecasting Challenge*.
  National Institute of Justice.
- PCRA. (2016). *Post-Conviction Risk Assessment Technical Manual*.
  Pennsylvania Commission on Sentencing.
- Skeem, J. L., & Lowenkamp, C. T. (2016). Risk, race, and recidivism:
  Predictive bias and disparate impact. *Criminology, 54*(4), 680–712.
