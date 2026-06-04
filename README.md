# ESTEEM-Egypt — Replication Package Documentation

**Thesis:** Egypt's Fuel Subsidy Reform and the Distributional Adequacy of Takaful and Karama Cash Transfers
**Model:** ESTEEM-Egypt — a 67-sector, 5-quintile dynamic input–output model adapted from Magacho & Spinola (2025)
**Author:** Menna Koura

This document explains how to reproduce every figure, table, and numerical result in the Methodology (Ch. 7), Data (Ch. 8), and Results (Ch. 9) chapters. It documents the two package files, the data flow from workbook to model, the equation system, the four scenarios, and the steps to run the code from a clean machine.

---

## 1. Package contents

| File | Role |
|------|------|
| `ESTEEM-EGY_by_Quintile` (Julia script, `.jl`) | The complete model: data load, calibration, symbolic ODE system, baseline solve, four scenarios, sensitivity grid, all plots and tables. |
| `ESTEEM-EGY ModelSheets.xlsx` | All calibration inputs across 14 sheets (IO core, household distribution, elasticities, subsistence, auxiliary reference tables). The script reads from this file only. |
| `Egypt_Subsistence_workbook.xlsx` | Audit workbook that **derives** the subsistence-share grid. Not read by the script — its output is pasted into the `Subsistence (c0_SH)` sheet of the model workbook. See §4.1. |

The script is the single entry point. It reads `ESTEEM-EGY ModelSheets.xlsx`, produces all `plot_*.png` files, and prints every quintile-level number used in the thesis text to the console.

---

## 2. Software environment

The model is written in **Julia** and depends on:

```julia
using Plots, DifferentialEquations, LinearAlgebra, XLSX, ModelingToolkit, Printf
```

Recommended setup (Julia ≥ 1.9):

```julia
import Pkg
Pkg.add(["Plots", "DifferentialEquations", "LinearAlgebra",
         "XLSX", "ModelingToolkit", "Printf"])
```

The DAE is assembled symbolically with `ModelingToolkit`, reduced with `structural_simplify`, and integrated with `Rosenbrock23()` (a stiff solver) over `t ∈ [0, 5]` with monthly output (`saveat = 1/12`), absolute tolerance `1e-8`, relative tolerance `1e-6`.

---

## 3. Running the model

1. Place the script and the workbook in the **same working directory**.
2. **Confirm the workbook filename matches line 14 of the script.** The script reads the workbook by name:

   ```julia
   path = pwd()
   EGYHH = XLSX.readxlsx(string(path, "/ESTEEM-EGY ModelSheets.xlsx"))
   ```

   The repository's workbook is `ESTEEM-EGY ModelSheets.xlsx`, which matches this line, so no change is needed. (If you ever rename the file, update line 14 to match exactly — filenames are case- and space-sensitive on Linux/macOS.)
3. Run the whole script: `julia ESTEEM-EGY_by_Quintile.jl` (or `include("...")` in the REPL).

The script will (in order): load and clean the IO data, print calibration diagnostics, calibrate the LES, build and simplify the symbolic system, solve the baseline, run the four scenarios, run the sensitivity analyses, and write every figure to PNG.

### 3.1 Sheet names match the code as-is

The 14 sheets in this workbook match the names the script references, including the two that are easy to get wrong:

- **`Own-Price Elasticities (ηxh)`** — read at `C2:G68` (columns C–G = Q1–Q5; rows 2–68 = the 67 sectors). The sheet name carries the `(ηxh)` suffix exactly as the code expects.
- **`Subsistence (c0_SH)`** — read at `I4:M70`. In this sheet, columns I–M are the "SHARE Q1…Q5" columns (the 67×5 subsistence-share grid ρ·shape), and rows 4–70 are the 67 IO sectors. After loading, confirm `size(C0_SH) == (67, 5)` and that values lie roughly in `[0.43, 0.80]` (necessity tilt: food/housing high, services low; Q1 column highest, Q5 lowest).

No sheet renaming or range edits are needed for this workbook. (If you ever swap in a workbook variant where these sheets were renamed or the subsistence grid relocated, you would need to reconcile the names/ranges — but that does not apply to the file documented here.)

---

## 4. Workbook structure: what each sheet supplies

The organising principle (Data chapter, §8.1) is that **levels** come from the 2016/17 IO table and **distributional shares** come from the 2019 SAM, applied to those levels. The table below lists every sheet, the exact range the script reads, and what it maps to in the model.

| Sheet | Range read | Symbol(s) | Meaning / units |
|-------|-----------|-----------|-----------------|
| `IO` | `C6:BQ72` | `ZZ` | 67×67 inter-industry flow matrix Z (’000 EGP). |
| `IO` | `BS6:BS72` + `BW6:BW72` | `cc` | Household final consumption (HH + NPISH), 67×1. |
| `IO` | `CE6:CE72` | `xx` | Total exports, 67×1. |
| `IO` | `BY6:BY72` | `iK` | Fixed capital formation, 67×1. |
| `IO` | `BV6:BV72` | `gg` | Government final consumption, 67×1. |
| `IO` | `CG6:CG72` | `mm` | Total imports, 67×1. |
| `IO` | `C85:BQ85` | `nn` | Employment by sector (persons); zeros guarded to 1. |
| `IO` | `C78:BQ78` − `C79:BQ79` | `tY_raw` | Net production tax = taxes on production − subsidies on production, 67×1. |
| `IO` | `C77:BQ77` | `WW_sec` | Compensation of employees (sectoral wage bill), 67×1. |
| `CQ` | `B2:F68` | `CQ` | Consumption by sector × quintile (67×5), rescaled to the IO HH total. |
| `POPQ` | `A2:E2` | `POPQ` | Population by quintile (’000); ≈ 20,329 each, equal fifths. |
| `THETAW` | `B2:F68` | `THETAW_raw` | Sectoral wage payments by quintile; **row-normalised** in code to wage shares Θʷ. |
| `APIQ` | `B5:F5` | `APIQ` | Profit-ownership shares by quintile (national, sums to ~1). |
| `SCQ` | `A8:E8` | `SCQ` | Social-contribution rates by quintile (fractions of wage income). |
| `Own-Price Elasticities (ηxh)` | `C2:G68` | `ETA_own` | Own-price elasticities ηʰᵢ by sector × quintile (Breisinger et al. 2018b). |
| `Trade Elasticities EZ` | `E3:E69`, `G3:G69` | `ηᴹ_vec`, `ηˣ_vec` | Baseline Eldeep–Zaki (2023) import/export elasticities (negative sign). |
| `Trade Elasticities BR` | `E6:E72` | `ηˣ_br` | Breisinger GTAP elasticities, used only in the sensitivity grid. |
| `Subsistence (c0_SH)` | `I4:M70` | `C0_SH` | 67×5 subsistence-share grid (SHARE Q1–Q5 columns; necessity pattern × Frisch level). |
| `Subsistence (frisch)` | — | — | Frisch/Lluch level calculation feeding the c0_SH grid (see §4.1); documentation only. |
| `Comparison` | — | — | Reference table: trade elasticities across all sources (documentation only). |
| `βᵖ Sector Mapping` | — | — | Documents the price-adjustment-speed assignment; the actual βᵖ vector is hard-coded in the script (see §6.2). |
| `SAM` | — | — | 2019 Social Accounting Matrix (CAPMAS & IFPRI 2021); source for the household-distribution shares above. Not read directly by the script (shares are pre-extracted into CQ/THETAW/APIQ/SCQ). |

Notes on construction:
- **Wage shares** Θʷ are formed by normalising each row of `THETAW_raw` to sum to 1 (share of sector *j*'s wage bill going to quintile *h*).
- **Profit shares** Θᵖ are derived (script §2b): the national `APIQ` vector is spread across sectors in proportion to gross operating surplus, corrected by a wage-distribution scaling factor, then row-normalised. This implements the SAM's uniform factor distribution (Data §8.5, limitation 2).
- **CQ** is read directly; mining records zero HH consumption in the SAM and is split equally across quintiles in the source sheet.

### 4.1 How the subsistence grid is built (audit workbook)

The `Subsistence (c0_SH)` sheet in the model workbook is **not estimated inside the model** — it is produced by a separate audit workbook (`Egypt_Subsistence_workbook.xlsx`) and pasted in. The model only consumes the final grid. The audit workbook is included for transparency and review; the chain is:

```
Egypt_Subsistence_workbook.xlsx  →  sheet "4_Egypt_Subsistence" (SHARE Q1–Q5 cols)
        →  pasted into  ESTEEM-EGY ModelSheets.xlsx  →  sheet "Subsistence (c0_SH)"  (I4:M70)
        →  read by the script as  C0_SH
```

The method combines an Egyptian **level** with a borrowed **shape** (Data §8.4):

- **Level** — the aggregate subsistence share per quintile comes from Egypt's *own* per-capita consumption via the Lluch et al. (1977) Frisch-parameter relationship $\omega = -36\,y^{-0.36}$, then $\gamma_{\text{agg}} = 1 + 1/\omega$, with $y$ in USD at **17 EGP/USD** (2019 rate, matching the Rwanda SAM year). This yields $\gamma_{\text{agg}}$ of 0.72, 0.68, 0.65, 0.61, 0.47 for Q1→Q5 (discretionary shares 27.9% → 53.5%).
- **Shape** — the relative split of subsistence *across sectors* is borrowed from a calibrated Rwandan LES (Mukashov et al. 2026, Step 4), collapsed from 15 household groups to 5 national quintiles by consumption-weighted averaging, then rescaled so each quintile's consumption-weighted average reproduces $\gamma_{\text{agg}}$.

Audit-workbook sheets (for review):

| Sheet | Content |
|-------|---------|
| `0_README` | Method statement (level vs. shape; what is Egyptian vs. borrowed). |
| `1_Rwanda_LES_Step4` | Rwanda subsistence (γ) shares for 15 household groups + consumption weights. |
| `2_Egypt_CQ Mapping` | Egypt inputs: consumption by sector × quintile (CQ), population, sector mapping. |
| `3_Calculations` | Frisch level, Rwanda 15→5 collapse, relative pattern. |
| `4_Egypt_Subsistence` | **Final 67×5 subsistence-share grid** (the SHARE Q1–Q5 columns are what gets pasted into the model). |
| `5_CrossCheck` | Egypt-vs-Rwanda necessity-ranking check; confirms the borrowed shape except utilities, where the model is a conservative lower bound (Data §8.4). |

> Only the necessity *distribution* is borrowed; the per-capita consumption, the basket, and the aggregate level are all Egyptian. An earlier approach that scaled Rwanda's shares by a uniform GDP-per-capita ratio was rejected as methodologically incorrect.

---

## 5. Data flow: from workbook to calibrated parameters

The script computes the base-year equilibrium quantities and prices before defining the dynamic system. Key steps (script §1–§4):

1. **Absorption and output:** `yA = rowsum(ZZ) + cc + gg + iK`; import propensity `sigmaM = mm ./ yA`; gross output `yy = (1 − sigmaM)·yA + xx`; technical coefficients `AA = ZZ · diag(yy)⁻¹`.
2. **Tax rates:** `tY = tY_raw ./ (yy + mm)` (net product-tax rate, negative where subsidised). Import tariffs `tM` are set to **zero** throughout (Data §8.5, limitation 6).
3. **Prices:** base producer prices normalised to 1; final price `pf = (1 + tY)·[(1 − sigmaM) + sigmaM·(1 + tM)]`; baseline markups `mu0` recovered from the cost identity; sectoral gross operating surplus `GOS_sec`.
4. **Quintile income:** wage income `DW = Θʷ′·WW_sec`; profit income `DPI`; disposable income `YDH_h = (1 − SCQ)·DW + DPI`; quintile CPIs `pCQ` weighted by each quintile's consumption basket.
5. **LES calibration (§4):** per-capita consumption `cQ = CQ / POPQ`; subsistence quantities `c0Q = C0_SH · cQ`; marginal budget shares `c1Q` recovered **residually** so the LES target reproduces observed base-year consumption exactly. The calibration check `max|CQ − Ctarg0| ≈ 0` must pass.

The aggregate subsistence shares $\gamma_{\text{agg}}$ that anchor this grid are derived in the audit workbook (§4.1) and reproduce the discretionary-share gradient (Q1: 27.9% → Q5: 53.5%) that drives the central consumption result in §9.4.

---

## 6. The model: equation blocks

The symbolic system is defined in script §5 (`eqs = vcat(...)`), simplified to an ODE system, and solved. Blocks correspond one-to-one with Methodology §7.4.

### 6.1 Block summary

| Block | Equations (thesis) | Script variables |
|-------|-------------------|------------------|
| Productive / output–inventory | 7.1–7.5 | `yᴰ, y, yᵉ, yᴬ, v, cₐ` |
| External (trade) | 7.6–7.8 | `x, σᴹ, m` |
| Household income distribution | 7.9–7.12 | `Wᴴ, Πᴴ, W, Π, YD` |
| Quintile LES consumption | 7.13, 7.22 | `pᶜᴴ, YDᴴ, Cᴴ` |
| Labour market & wages | 7.14–7.16 | `n, wᵈ, w` |
| Price formation & inflation | 7.17–7.23 | `UC→pᵈ, pᵇ, μ, pᶠ, pᶜ, π, πʸ` |
| Government & fiscal | 7.24–7.29 | `Tʸ, Tᴹ, SC, SG, GDP, ST, STᴴ` |

A few implementation details worth noting for replication:
- Consumption (eq. 7.13) and trade (7.6–7.8) use `max(·, 1e-8)` floors inside the power terms to avoid `DomainError` from negative bases; these guards never bind at the reported solution.
- GDP (eq. 7.27) is measured at **basic prices** and enters only as the scaling denominator in the transfer rule; it is not an input to the Leontief block.
- The exchange rate `eᴺ` is fixed at **18 EGP/USD** in the parameter dictionary and world prices `pᵂ = 1/18`, so `pᵂ·eᴺ = 1` in the base year. (The thesis text §7.4.3 cites 17.8 as the 2017 post-float average; the code uses 18. The difference is a normalisation of the base-year world price and does not affect the ratios-to-baseline that all results report.)

### 6.2 Behavioural parameters (script §6)

Set in the parameter dictionary `p`. Values match Methodology Table 7.2 / 7.3:

- **Output-expectation speed βʸ:** 1.0 for slow-adjusting sectors (agriculture 1–2, extraction 3–4, petroleum refining/electricity/water 14,29,30); 3.0 for manufacturing and services.
- **Price-adjustment speed βᵖ:** sector-specific, hard-coded (`βᵖ_vec`) — agriculture/mining/transport 6.0; manufacturing/construction 3.0; energy & water 1.5; real estate 2.0; accommodation 1.0; ICT & education 0.2; default (other services) 2.0. The `βᵖ Sector Mapping` sheet documents this assignment for reference.
- **βᶜ = βʷ = 1.0** (one-year benchmark); **μ₁ = 0.1**; **γⁿ = λⁿ = 1.0**.
- **Trade elasticities:** baseline uses the Eldeep–Zaki vectors (`ηˣ_vec`, `ηᴹ_vec`); the base-year `ζˣ`, `ζᴹ` propensities are recovered by inverting the trade equations at observed flows.

---

## 7. Initial conditions and baseline validation (script §7–§8)

The system is initialised at the observed base-year equilibrium (`u0`), so a no-shock solve should be **stationary**. The script prints the validation checks the thesis §7.8 relies on:

- Aggregate output, aggregate and quintile CPIs, real disposable income by quintile, the exchange rate, and the NX/GDP ratio all remain flat to solver tolerance.
- The saving–investment identity `SP + SG − NX − Inv = 0` (eq. 7.31) holds to machine precision (max residual ≈ 5.5×10⁻¹⁰ relative to GDP).

Only after baseline stationarity is confirmed are the shock scenarios applied, so any movement is attributable to the reform.

---

## 8. The shock and the four scenarios (script §9)

### 8.1 The subsidy shock

The net product-tax rate on **sector 14 (petroleum refining)** is scaled toward cost recovery, retaining the LPG share (19.07%, from Breisinger et al. 2018b):

```julia
tY_new[14] = tY[14] * 0.190734179
```

This moves `tY[14]` from **−0.316 to −0.060** (Δ = +0.256), reproducing eq. 7.30 and the §9.1 impact (petroleum final price +≈37% on impact). All other sectors are unchanged; the shock propagates through the IO cost linkage.

### 8.2 Scenario definitions

Each scenario sets `(στˢ, δˢ)` in the transfer rule (eq. 7.28); the transfer is directed to the bottom 40% via `στᴴ = [0.5, 0.5, 0, 0, 0]`.

| Scenario | `στˢ` | `δˢ` | Transfer | Meaning |
|----------|-------|------|----------|---------|
| **S0** | 0 | 0 | none | Pure shock, no compensation. |
| **S1** | 1 | 1 | full dynamic fiscal saving (≈ 2.86% of GDP) | Normative upper bound. |
| **S2** | 0.004 | 0 | 0.4% of GDP | IMF 4th-Review structural benchmark. |
| **S3** | 0.0025 | 0 | 0.25% of GDP | Actual average TKP scale, 2017–2024. |

`δˢ = 1` makes the transfer equal the dynamic rise in net product-tax revenue from the reform; `δˢ = 0` makes it a fixed fraction of nominal GDP.

### 8.3 Scenario solves reproduce Results §9.5

Each scenario is solved as a separate `ODEProblem` (`sol_s0`…`sol_s3`) with the same `u0` and solver settings. The script prints, at t = 5: real income by quintile (Table 9.5), the bottom-40% income ratio and compensation-adequacy (Tables 9.4, 9.6), CPI by quintile, aggregate output, and the consumption decomposition (Table 9.3). Expected headline values: B40 income at t=5 of 0.904 (S0), 1.043 (S1), 0.924 (S2), 0.916 (S3); gap closed 0%/144.7%/21.0%/13.1%.

---

## 9. Sensitivity analysis (script §11 and full grid)

Run on **S2** as the central scenario. Three parameter groups are varied:

1. **Adjustment speeds** βᵖ (fast = 6.0 / baseline / slow = 0.5) and βʸ (fast / baseline / slow) — Table 9.7, Panel A.
2. **Markup sensitivity** μ₁ ∈ {0.0, 0.1, 0.5} — Panel B.
3. **Trade elasticities** — uniform −0.7, Eldeep–Zaki (baseline), Cockburn −2.0, Griffin −2.5, Breisinger GTAP — Panel C.

The **full 5×3 grid** (trade elasticity × βᵖ) is run separately to produce Figure 9.8 (`plot_sensitivity_grid.png`). The qualitative check counts how many of the 15 cells leave Q1 below baseline — the result that **all 15 do** is the robustness claim in §9.6. When changing trade elasticities, the script also recomputes the `ζˣ`, `ζᴹ` base-year propensities consistently for each elasticity set.

---

## 10. Figure and table → output-file map

| Thesis item | Script output |
|-------------|---------------|
| Fig. 9.1 Transmission (S0) | `plot_TX_transmission_mechanism.png` |
| Fig. 9.2 Output & prices by group | `plot_prop_3_production.png` (+ `plot_J`, `plot_K` grouped bars) |
| Fig. 9.3 Trade by group | `plot_prop_2_trade.png` |
| Fig. 9.4 Household outcomes (S0) | `plot_prop_1_households.png` |
| Fig. 9.5 Consumption by quintile, all scenarios | `plot_L_consumption_quintiles.png` |
| Fig. 9.6 Bottom-40% income, all scenarios | `plot_A2_bottom40_four_scenarios.png` |
| Fig. 9.7 Real income by quintile, all scenarios | `plot_B_quintiles_all_scenarios.png` |
| Fig. 9.8 Sensitivity grid | `plot_sensitivity_grid.png` |
| Top-15 sector price / output charts | `plot_H_top15_price.png`, `plot_I_top15_output.png` |
| Petroleum price path | `plot_M_petroleum_price.png` |
| Baseline diagnostics (Appendix) | `plot_appendix_baseline_stability.png` |
| Tables 9.1–9.7 | printed to console (`TABLE 1`…`TABLE 4`, plus verification blocks) |

The console "VERIFICATION OUTPUT" section prints every quintile-level ratio at t = 0,1,2,3,4,5 used in the text, so each in-text number can be traced to a printed value.

---

## 11. Known data artefacts (Results §9.2.3, §9.6.2)

Ten sectors have under-populated intermediate-input columns in the CAPMAS table, producing artefactually high markups and exaggerated own-sector responses (most visibly the −67% collapse in Wood products). They are **retained for Leontief consistency** but flagged and excluded from substantive interpretation. The script comment (≈ line 981) lists them as sectors **3, 11, 13, 27, 28, 34, 37, 47, 52, 65**. They account for ≈ 4% of GDP and appear sparsely in the IO matrix, so they do not propagate materially into aggregate or quintile-level outcomes.

---

## 12. Reproducibility checklist

- [ ] Workbook `ESTEEM-EGY ModelSheets.xlsx` is in the same folder as the script and its name matches line 14.
- [ ] Sheets present with expected names — including `Own-Price Elasticities (ηxh)` and `Subsistence (c0_SH)` — so no renaming is needed for this workbook.
- [ ] Subsistence read `["Subsistence (c0_SH)"]["I4:M70"]` returns a 67×5 numeric grid; confirm `size(C0_SH) == (67,5)`.
- [ ] Julia packages installed.
- [ ] Baseline checks pass: output/CPI/income flat; S=I residual ≈ 1e-10; LES `max|CQ − Ctarg0| ≈ 0`; `tY[14]: −0.316 → −0.060`.
- [ ] Scenario console output matches Table 9.5 / 9.6; sensitivity grid reports 15/15 cells below baseline.

Once these pass, every figure and table in Chapters 7–9 regenerates from a single run.
