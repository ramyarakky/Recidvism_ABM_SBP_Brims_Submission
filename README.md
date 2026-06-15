# Recidivism ABM — Phase 1: Three-Stage OAT Calibration

**Project:** An Agent-Based Model of Recidivism and Fairness  
**Conference:** SBP-BRiMS 2026 — Social, Cultural, and Behavioral Modeling  
**Author:** Ramya Rakkiappan, PhD Candidate, Computational Science, George Mason University  
**Supervisor:** Dr. Hamdi Kavak  

---

## Overview

Recidivism prediction models are widely deployed in United States criminal justice
settings to inform supervision, parole, and sentencing decisions. Despite their
influence, the internal mechanisms driving these predictions remain difficult to
validate for fairness , particularly whether risk scores produce structurally
equitable outcomes across demographic groups.

This repository implements an **agent-based model (ABM)** of post-release recidivism
in the U.S. justice system, developed as part of a PhD dissertation on algorithmic
fairness at George Mason University. The model simulates the post-release trajectories
of a synthetic cohort of formerly incarcerated individuals, capturing offender
heterogeneity, supervision dynamics, peer influence, and criminal justice interventions
under race and gender neutral baseline assumptions.

### Why an Agent-Based Model?

Statistical recidivism models (e.g., logistic regression, gradient boosting) optimize
for predictive accuracy but do not represent the mechanisms through which structural
disparities emerge. An ABM allows explicit specification of those mechanisms such as
supervision intensity, risk-tier assignment, offense trajectories by enabling
counterfactual experiments that ask not just *who* reoffends but *why* the system
produces the outcomes it does.

### What Phase 1 Does

Phase 1 establishes the **empirically calibrated baseline** model that reproduces
known national and tier-stratified rearrest statistics before any bias is introduced.
This baseline is essential: it ensures that any disparities observed in later phases
are attributable to the mechanisms under study, not to a mis-specified starting point.

Calibration proceeds in three sequential stages using one-at-a-time (OAT) parameter
sweeps across 11 parameters (3 fixed, 8 estimated), validated against two independent
empirical sources:

- **BJS NCJ 250975** (Alper et al., 2018) — national aggregate and per-offense
  rearrest rates at 3, 6, and 9 years post-release
- **Federal Probation 87(2)** (Johnson, 2023) — PCRA risk-tier-stratified rearrest
  rates for Low, Low-Moderate, Moderate, and High risk groups

The three stages proceed in a fixed order — each stage locks its parameters before
the next begins:

```
Stage 1 → BJS National Aggregate      (α, δ_s3, δ_s6 + 3 fixed desistance ratios)
Stage 2 → PCRA Tier-Stratified        (γ — risk contrast strength)
Stage 3 → BJS Per-Offense Rearrest    (o_v, o_d, o_p, o_o — offense hazard shifts)
```

The calibrated model achieves a mean absolute error (MAE) below 2.5 percentage points
across all follow-up windows, providing a validated foundation for fairness assessment
in subsequent phases.

### Dissertation Context

| Phase | Focus | Status |
|-------|-------|--------|
| Phase 1 | Empirical calibration — this repository | ✅ Complete |
| Phase 2 | Bias injection — structural disparities by race and gender | 🔄 In progress |
| Phase 3 | Fairness interventions and counterfactual analysis | 🔲 Future work |

---

## Model Configuration

| Parameter | Value |
|-----------|-------|
| Initial agents | 3,000 |
| Warm-up period | 144 months (12 years) |
| Study period | 108 months (9 years) |
| Monthly intake | 10 agents |
| Peer influence | Enabled |
| Mode | Realistic |
| Bias factor | 0.0 (Phase 1 — no bias injected) |
| Runs per sweep point | 100 simulations |

---

## Three-Stage Calibration

### Stage 1 — BJS National Aggregate Calibration

**Goal:** Match cumulative rearrest rates at 3, 6, and 9 years to BJS NCJ 250975.

**What it does:** Calibrates supervision-related parameters governing how intensely
agents are monitored after release and how that monitoring decays over time. Higher
supervision intensity increases violation detection probability; decay multipliers
reduce that intensity after years 3 and 6, reflecting empirically observed declines
in supervision contact (Petersilia, 2003).

**Calibration targets:**

| Window | BJS Target | Source |
|--------|-----------|--------|
| 3-year | 68.4% | Alper et al. (2018), BJS NCJ 250975 |
| 6-year | 79.4% | Alper et al. (2018), BJS NCJ 250975 |
| 9-year | 83.4% | Alper et al. (2018), BJS NCJ 250975 |

> **Diagnostic anchor (not a calibration target):** 1-year rate = 43.9%

**Estimated parameters:**

| Symbol | Parameter | Baseline | Calibrated | Sweep Range | Step |
|--------|-----------|----------|------------|-------------|------|
| α | Supervision Monitoring Intensity | 1.000 | **1.120** | [1.00, 1.20] | 0.02 |
| δ_s3 | Supervision Decay — Years 3–6 | 1.000 | **0.990** | [0.60, 0.99] | ~0.032 |
| δ_s6 | Supervision Decay — Years 6–9 | 1.000 | **0.400** | [0.20, 0.70] | 0.05 |

**Fixed parameters (BJS-anchored, not estimated):**

| Symbol | Parameter | Fixed Value | Derivation |
|--------|-----------|-------------|------------|
| dr1 | Desistance ratio — Years 1–3 | 0.524 | q₂₃/q₁ from BJS hazard decomposition |
| dr3 | Desistance ratio — Years 3–6 | 0.500 | q₂/q₁ from BJS hazard decomposition |
| dr6 | Desistance ratio — Years 6–9 | 0.508 | q₃/q₂ from BJS hazard decomposition |

**Loss function:** Mean absolute error (MAE) across BJS aggregate targets at 3, 6,
and 9 years. Each window is the primary loss target for its corresponding parameter
(α → all three windows; δ_s3 → 6-year primary; δ_s6 → 9-year primary).

**Stage 1 results:**

| Window | Uncalibrated | Calibrated | BJS Target | MAE |
|--------|-------------|------------|------------|-----|
| 3-year | 65.8% | 70.2% | 68.4% | 0.018 |
| 6-year | 78.0% | 79.9% | 79.4% | 0.005 |
| 9-year | 80.8% | 81.5% | 83.4% | 0.019 |

---

### Stage 2 — PCRA Tier-Stratified Calibration

**Goal:** Match 3-, 6-, and 9-year rearrest rates within each PCRA risk tier
to Johnson (2023) Federal Probation empirical targets.

**What it does:** Introduces a risk-contrast parameter γ, which scales each agent's
baseline hazard by sᵢ^γ, where sᵢ ∈ (0,1] is the agent's normalized PCRA score.
At γ = 0 all tiers share identical hazard dynamics; increasing γ widens separation
across tiers. Aggregate loss alone is insensitive to γ, as cross-tier errors cancel;
tier-stratified calibration provides the necessary additional constraint.

**Calibration targets (3-year):**

| PCRA Tier | Target | Source |
|-----------|--------|--------|
| Low | 46.2% | Johnson (2023), Federal Probation 87(2), Table 6 |
| Low-Moderate | 72.0% | Johnson (2023), Federal Probation 87(2), Table 6 |
| Moderate | 84.5% | Johnson (2023), Federal Probation 87(2), Table 6 |
| High | 91.0% | Johnson (2023), Federal Probation 87(2), Table 6 |

**Estimated parameter:**

| Symbol | Parameter | Baseline | Calibrated | Sweep Range | Step |
|--------|-----------|----------|------------|-------------|------|
| γ | Risk Contrast Strength | 0.000 | **1.000** | [0.75, 1.50] | 0.05 |

**Loss function:** Mean absolute error across 12 cells (4 PCRA tiers × 3 follow-up
windows: 3, 6, and 9 years post-release).

**Stage 2 results (3-year, calibrated vs. target):**

| PCRA Tier | Uncalibrated | Calibrated | Target | Δ |
|-----------|-------------|------------|--------|---|
| Low | 65.5% | 52.3% | 46.2% | +6.1 pp |
| Low-Moderate | 66.0% | 69.4% | 72.0% | −2.6 pp |
| Moderate | 65.9% | 82.6% | 84.5% | −1.9 pp |
| High | 65.5% | 92.0% | 91.0% | +1.0 pp |

Mean |Δ| reduced from 17.35 pp (uncalibrated) to 2.90 pp (calibrated).

---

### Stage 3 — BJS Per-Offense Rearrest Calibration

**Goal:** Match cumulative rearrest rates by offense type (Violent, Drug, Property,
Other/Public Order) at 3, 6, and 9 years to BJS NCJ 250975 Table 7.

**What it does:** Introduces four offense-specific log-odds shifts applied to the
hazard equation after Stage 1–2 parameters are locked. Each shift adjusts group-level
rearrest odds relative to the tier baseline. Parameters are swept independently; the
sweep range for each offense is informed by the baseline residual gap observed after
Stage 1–2 calibration.

**Calibration targets and baseline gaps (3-year):**

| Offense | Target | Baseline Gap | Source |
|---------|--------|-------------|--------|
| Violent | 62.2% | +8.6 pp over target | Alper et al. (2018), NCJ 250975 Table 7 |
| Drug | 68.6% | −0.1 pp under target | Alper et al. (2018), NCJ 250975 Table 7 |
| Property | 75.0% | −8.9 pp under target | Alper et al. (2018), NCJ 250975 Table 7 |
| Other/Public Order | 65.0% | +8.1 pp over target | Alper et al. (2018), NCJ 250975 Table 7 |

**Estimated parameters:**

| Symbol | Parameter | Baseline | Calibrated | Sweep Range | Step |
|--------|-----------|----------|------------|-------------|------|
| o_v | Violent offense shift | 0.000 | **−0.300** | [−0.40, +0.05] | ~0.05 |
| o_d | Drug offense shift | 0.000 | **+0.050** | [−0.15, +0.20] | ~0.05 |
| o_p | Property offense shift | 0.000 | **+0.600** | [+0.20, +0.80] | 0.05 |
| o_o | Other/Public Order shift | 0.000 | **−0.400** | [−0.40, +0.05] | ~0.05 |

**Loss function:** Mean absolute error across 12 cells (4 offense types × 3 follow-up
windows). Each offense is swept independently with all other parameters locked.

---

## Full Parameter Summary

| Stage | Symbol | Parameter | Baseline | Calibrated |
|-------|--------|-----------|----------|------------|
| 2 | γ | Risk Contrast Strength | 0.000 | 1.000 |
| 1 | α | Supervision Monitoring Intensity | 1.000 | 1.120 |
| 1* | dr1 | Desistance Years 1–3 [fixed] | — | 0.524 |
| 1* | dr3 | Desistance Years 3–6 [fixed] | — | 0.500 |
| 1* | dr6 | Desistance Years 6–9 [fixed] | — | 0.508 |
| 1 | δ_s3 | Supervision Decay 3–6 yr | 1.000 | 0.990 |
| 1 | δ_s6 | Supervision Decay 6–9 yr | 1.000 | 0.400 |
| 3 | o_v | Violent offense shift | 0.000 | −0.300 |
| 3 | o_d | Drug offense shift | 0.000 | +0.050 |
| 3 | o_p | Property offense shift | 0.000 | +0.600 |
| 3 | o_o | Other/Public Order shift | 0.000 | −0.400 |

*Fixed at BJS hazard decomposition anchor — not swept.

---

## Output Files

```
OAT_Calibrate_BJS_PCRA_Offense_Output/
│
├── baseline.json                        # Uncalibrated run results
├── recommended_params.json              # All 11 final calibrated parameters
│
├── sweep_alpha.csv                      # Stage 1: α sweep
├── sweep_smi_decay3y.csv                # Stage 1: δ_s3 sweep
├── sweep_smi_decay6y.csv                # Stage 1: δ_s6 sweep
├── sweep_gamma.csv                      # Stage 2: γ sweep
├── sweep_oshift_violent.csv             # Stage 3: o_v sweep
├── sweep_oshift_drug.csv                # Stage 3: o_d sweep
├── sweep_oshift_property.csv            # Stage 3: o_p sweep
├── sweep_oshift_pubord.csv              # Stage 3: o_o sweep
│
├── FINAL_calibration_summary.png        # Parameter table + aggregate bar chart
├── THREE_WAY_comparison.png             # 4-panel: bars / error bars / MAE table / lollipop
├── CalibrationSummary.png               # Clean single-panel aggregate summary
├── STAGE3_offense_validation.png        # Per-offense baseline vs. calibrated vs. BJS
├── chart3_cumulative_by_offense.png     # Cumulative trajectories by offense type
│
├── equifinality.png                     # Tier rates: uncalibrated vs. calibrated vs. PCRA
├── tier_chart1_before_after_3yr.png     # Stage 2 before/after by tier
├── tier_chart2_trajectories.png         # Per-tier trajectories 3/6/9 yr
├── tier_chart3_gap_heatmap.png          # Tier × window gap heatmap
├── tier_chart4_dashboard.png            # Aggregate + tier dashboard
├── tier_composition.png                 # PCRA tier composition donut
│
├── seed_strip.png                       # Per-seed strip plot
├── seed_strip_with_within_run_ci.png    # Two-layer uncertainty view
├── seed_convergence.png                 # Running mean convergence across seeds
└── seed_mcse_errorbar.png              # 95% CI of the mean vs. BJS target
```

---

## Running the Code

### Full calibration (runs all simulations — slow)
```bash
python OAT_Calibrate_BJS_PCRA.py
```

### Replot all charts from existing results (fast — no simulations)
```bash
python OAT_Calibrate_BJS_PCRA.py --replot
```

### Force re-run ignoring checkpoints
```bash
python OAT_Calibrate_BJS_PCRA.py --rerun
```

### Specify number of parallel workers
```bash
python OAT_Calibrate_BJS_PCRA.py --cores 8
```

> Core detection is automatic via `psutil` if installed. Without it, the script
> falls back to `multiprocessing.cpu_count() // 2 - 1`.

---

## Dependencies

```
python >= 3.9
mesa
numpy
pandas
matplotlib
tqdm
psutil          # optional — for automatic core detection
```

```bash
pip install mesa numpy pandas matplotlib tqdm psutil
```

---

## References

- Alper, M., Durose, M. R., & Markman, J. (2018). *2018 Update on Prisoner Recidivism:
  A 9-Year Follow-Up Period (2005–2014)*. BJS NCJ 250975.
- Johnson, J. L. (2023). Predicting reentry success using the Post-Conviction Risk
  Assessment. *Federal Probation, 87*(2), Table 6.
- Petersilia, J. (2003). *When Prisoners Come Home: Parole and Prisoner Reentry*.
  Oxford University Press.
- Langan, P. A., & Levin, D. J. (2002). *Recidivism of Prisoners Released in 1994*.
  BJS NCJ 193427.

---

## Citation

```
Rakkiappan, R., & Kavak, H. (2026). An Agent-Based Model of Recidivism.
In Proceedings of SBP-BRiMS 2026.
```
