# F1 Grid Position: A Bayesian Inference Study

![R](https://img.shields.io/badge/R-276DC3?logo=r&logoColor=white) ![Stan](https://img.shields.io/badge/Stan-cmdstanr-B2001D) ![Bayesian](https://img.shields.io/badge/Bayesian-hierarchical%20model-4B8BBE) ![MCMC](https://img.shields.io/badge/Inference-MCMC%20%2B%20VI-2E8B57)

Does starting further back on the grid cost a Formula 1 driver more time at the finish, and does that cost depend on the circuit? This project answers both questions with Bayesian models fit in Stan.

**Result:** doubling a driver's grid position increases their expected gap to the race winner by about **1.6 times**. The effect varies modestly but really by circuit, from about 1.5 times at Hockenheimring to 1.75 times at Sochi.

📄 **Full report:** [report.pdf](report.pdf)

## Data
Race results for 2019 to 2024 from the [f1dataR](https://github.com/SCasanova/f1dataR) package: **1,464 finishers across 31 circuits**.

- **Response:** `gap_seconds`, the finishing time gap to the race winner. It's never negative and is strongly right-skewed.
- **Predictor:** starting grid position (1 to 20), log-transformed, since the relationship with the gap is non-linear.

The dataset is saved as `full_data.csv`, so you don't need to download it again.

## Models

**Model 1: one grid effect for all circuits**

    gap_i ~ Exponential(λ_i)
    λ_i = exp(α + β · log(grid_i))
    α ~ N(-2.5, 1),  β ~ N(-0.5, 0.5)

The exponential likelihood fits the skewed, non-negative gaps, and the priors were chosen from the exploratory analysis and checked with prior predictive simulation.

**Model 2: a grid effect for each circuit (hierarchical)**

    β_j ~ N(μ_β, σ_β),  j = 1, ..., 31
    μ_β ~ N(-0.5, 0.5),  σ_β ~ Exponential(1)
    λ_i = exp(α + β_j[i] · log(grid_i))

Each circuit gets its own grid effect, but circuits with only a few races borrow strength from the rest. This is called partial pooling: it keeps a three-race sample from being treated the same as a fifteen-race one.

Both models were fit with **MCMC** (5 chains) and **variational inference**: mean-field VI for Model 1, and full-rank VI for Model 2, since its 31 circuit effects are strongly correlated.

## Validation
- **Mixing and convergence:** trace plots, rank histograms, effective sample sizes and R-hat
- **Calibration on held-out data:** 80% credible intervals checked against 50 randomly held-out results
- **Simulation-based checks:** coverage on data simulated from the prior, plus an exact invariance test (Kolmogorov-Smirnov p = 0.91 for Model 1 and 0.37 for Model 2, so no sign of implementation bugs)
- **Posterior predictive checks** comparing simulated and observed gaps
- **MCMC vs VI comparison** of estimates and intervals

## Results

| | Model 1 | Model 2 (hierarchical) |
|---|---|---|
| Grid effect (β or μ_β) | -0.70 | -0.70 |
| Circuit variation (σ_β) | n/a | 0.09 |
| MCMC coverage of 80% intervals | 82% | 84% |
| VI coverage of 80% intervals | 80% | 80% |

- **Overall effect:** β = -0.70, so doubling grid position multiplies the expected gap by 2^0.70 ≈ 1.62. Both models agree exactly on the global effect.
- **Circuit differences are real but modest** (σ_β = 0.09). Sochi has the steepest effect (×1.75 per doubling), where overtaking is hard, and Hockenheimring the flattest (×1.50), where overtaking is easier.
- **Shrinkage works as intended:** circuits with few races (Sochi, Baku, Zandvoort, Interlagos) are pulled most toward the overall average, while data-rich circuits like Silverstone and Spa barely move.

![Shrinkage of circuit-level grid effects](plots/shrinkage_Hierarchical/shrinkage_plot_Hierarchical.png)

## Limitations
- The exponential likelihood ties the spread to the mean, so both models underestimate variation in the 30 to 100 second range. A Gamma likelihood would fix this.
- Only grid position is used. Weather, safety cars and car performance all affect gaps too.
- Circuits share a single intercept. Circuit-specific intercepts would be a natural next step.

## How to run
You need R and [CmdStan](https://mc-stan.org/cmdstanr/) (installed through the `cmdstanr` package).

1. Install the R packages:

       install.packages(c("tidyverse", "bayesplot", "distr", "f1dataR"))
       install.packages("cmdstanr", repos = c("https://stan-dev.r-universe.dev", getOption("repos")))
       cmdstanr::install_cmdstan()

2. Clone the repository and open `stat-405-final-project.Rproj` in RStudio.
3. Knit the R Markdown files in this order:

| File | What it does |
|---|---|
| `eda.Rmd` | Exploratory analysis of gaps, grid positions and circuits |
| `simple_model_clean.Rmd` | Model 1: fitting, diagnostics and calibration |
| `hierarchical_models.Rmd` | Model 2: fitting, diagnostics, calibration and shrinkage |
| `report.Rmd` | Final report |

The Stan models are in `simple_model_exp.stan` and `hierarchical_model_exp.stan`, and compile automatically the first time they run.

<details>
<summary>Rebuilding the dataset from scratch (slow)</summary>

`full_data.csv` is already included. To regenerate it with f1dataR:

    library(f1dataR)
    library(tidyverse)

    full_data <- map_dfr(2019:2024, function(year) {
      schedule <- load_schedule(season = year)
      map_dfr(seq_len(nrow(schedule)), function(round) {
        tryCatch(
          load_results(season = year, round = round) %>%
            mutate(
              season = year,
              round = round,
              race_name = schedule$race_name[round],
              circuit = schedule$circuit_id[round],
              top_speed_kph = as.numeric(top_speed_kph),
              fastest_rank = as.integer(fastest_rank)
            ),
          error = function(e) {
            message("Skipping season ", year, " round ", round, ": ", e$message)
            NULL
          }
        )
      })
    })

    write_csv(full_data, "full_data.csv")

</details>

## Authors
Shahrukh Islam Prithibi and Khue Do.

## Reference
van Kesteren, E.-J., & Bergkamp, T. (2023). Bayesian analysis of Formula One race results: disentangling driver skill and constructor advantage. *Journal of Quantitative Analysis in Sports, 19*(4), 273-293. https://doi.org/10.1515/jqas-2022-0021
