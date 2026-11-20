# Provenance-Aware Belief Aggregation in Multi-Agent Communication

[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-3776AB)](requirements.txt)
[![License: MIT](https://img.shields.io/badge/License-MIT-green)](LICENSE)

Code and data for the paper *"Provenance-Aware Belief Aggregation in
Multi-Agent Communication Under Convention Heterogeneity"* (Jayant Dubey).
The manuscript is at [paper/main.pdf](paper/main.pdf) (17 pages, LaTeX
sources in `paper/sections/`).

A plain-language walkthrough is in
[EXPLAINED_FOR_BEGINNERS.md](EXPLAINED_FOR_BEGINNERS.md).

## Problem

Multi-agent systems commonly aggregate messages using source-reliability
weights. That fails when sources use different reporting conventions —
mappings from internal readings to messages that can invert, condition on
context, or otherwise transform the underlying signal. A weight on a source
cannot correct for a source that reports the opposite of what it measured.

Concretely: if one agent reports `NOT` of what it measured and another
reports the identity, averaging their messages is wrong no matter how the
two are weighted. The fix has to decode *how* each source reports before
combining *what* they report.

## Method

A finite-grid Bayesian latent-convention aggregator: each source's
reporting convention is treated as a latent variable and decoded jointly
with the underlying reading, instead of weighting messages directly. The
aggregator enumerates a fixed grid over convention stability (ρ) and
reliability (r) in full — no sampling, no randomness in the aggregator
itself — and combines the per-source posteriors into a single belief.

The repository also contains structural limitation theorems and their
proofs (`paper/sections/theorems.tex`):

- **Nonnegative-weight insufficiency** — no nonnegative weighting of
  messages can recover the truth under convention heterogeneity.
- **Scalar insufficiency under context-dependent conventions** — a single
  scalar per source cannot capture a convention that conditions on
  context.
- **Equal-reliability optimum** and **posterior consistency** — the
  conditions under which the Bayesian aggregator is optimal, and that it
  converges as evidence accumulates.

Ten experiments cover the convention-stability sweep (A), representation
ablations (B), support-size scaling (C), reliability heterogeneity (D),
multi-agent consensus (E), an LLM communication benchmark (F), a
real-human-judgment CIFAR-10H study, candidate-set misspecification (G),
correlated sources (H), and the stability boundary (I) — when per-source
decoding helps versus hurts — plus PVI-REAL (§6.13): controlled panels,
CrowdTruth RelEx/RelDir real-convention validation with a calibrated
decoder, and an adversarial failure map.

Baselines: content-only, unsigned scalar, signed scalar,
context-conditioned scalar, context-dependent logistic regression
(CtxLogReg), attention calibrator, mean-pool calibrator, EM decoder (MAP
point estimate), population-average mapping, Bayesian aggregator, and
reliability-weighted oracle.

## Results

Headline results banked over 190 seeds (A: 100; B/C/E: 30 each), mean ±
95% confidence interval:

| Experiment | Accuracy |
|---|---|
| A (ρ = 1.0) | 0.7576 ± 0.0021 |
| B | 0.7590 ± 0.0044 |
| C (K = 50) | 0.8204 ± 0.0029 |
| E (12 agents) | 0.9489 ± 0.0114 |

- **Multi-agent consensus (12 agents):** +37.5 percentage points over
  naive averaging, within 4.3 points of the oracle.
- **Stability boundary (Experiment I):** at unstable conventions
  (ρ = 0.5) a population-average mapping beats per-source Bayesian
  decoding (+0.0196 at K = 10, 95% CI [+0.0151, +0.0241], dz = +1.63;
  the edge shrinks with support and vanishes at K = 50). At ρ ≥ 0.6
  decoding wins decisively (−0.2803 at ρ = 1.0, dz = −12.74). The
  boundary is sharp and prescriptive: decode only when conventions are
  stable enough.
- **Reported honestly:** the best learned calibrator beats Bayes in B
  (MetaAttn+Dec, +0.0649 ± 0.0028, paired, Cohen dz = +8.79).
- **Calibration:** the Bayesian posterior scores best probabilistically
  (NLL, Brier, ECE) but grows overconfident as conventions stabilise —
  reported as a limitation, not hidden.
- **Real-world anchor:** Anthropic's Sonnet 5.5 system card reports
  FrontierCode scores of 52.1% at `xhigh` effort and 46.2% at `max`
  effort — more reasoning effort scoring worse, a deployed echo of the
  paper's noise-not-signal caution (cited in the paper, § limitations).

### Experiment A — convention-stability sweep (100 seeds)

Accuracy, mean ± 95% CI, from `results/analysis_n100.txt`. Bayes pulls
away from every baseline as conventions stabilise; at ρ = 0.5 the
population-average mapping (NoSupport) is still ahead of it.

| ρ | Bayes | MetaAttn | NoSupport | SignedScalar |
|---|---|---|---|---|
| 0.5 | 0.6317 ± 0.0023 | 0.6206 ± 0.0028 | 0.6485 ± 0.0029 | 0.6173 ± 0.0023 |
| 0.6 | 0.6376 ± 0.0021 | 0.6243 ± 0.0027 | 0.6174 ± 0.0026 | 0.5985 ± 0.0023 |
| 0.7 | 0.6554 ± 0.0019 | 0.6350 ± 0.0022 | 0.5825 ± 0.0023 | 0.5892 ± 0.0025 |
| 0.8 | 0.6809 ± 0.0019 | 0.6492 ± 0.0027 | 0.5471 ± 0.0025 | 0.5896 ± 0.0021 |
| 0.9 | 0.7161 ± 0.0023 | 0.6756 ± 0.0030 | 0.5160 ± 0.0029 | 0.6049 ± 0.0027 |
| 1.0 | 0.7576 ± 0.0021 | 0.7012 ± 0.0046 | 0.4831 ± 0.0037 | 0.6209 ± 0.0022 |

Bayes also has the best negative log-likelihood at every ρ (0.6342 at
ρ = 0.5 down to 0.5480 at ρ = 1.0), against 0.68–0.69 for NoSupport at
high ρ — the posterior is better calibrated as well as more accurate.

![Convention-stability sweep, 100 seeds](figures/fig1_rho_sweep_n100.png)

### Experiment F — LLM communication benchmark (20 trials)

Models' private beliefs are supplied rather than elicited, so this is a
controlled communication test, not a debate. Means ± 95% CI computed
from the banked `results/llm_benchmark.csv` (normal approximation over
the 20 trials). Everything is near ceiling here — the informative
comparison is against naive averaging, which trails by ~17 points.

| Method | Accuracy |
|---|---|
| SignedScalar | 0.9950 ± 0.0067 |
| ContextConditioned | 0.9900 ± 0.0090 |
| Bayes | 0.9875 ± 0.0097 |
| Oracle (reliability-weighted) | 0.9800 ± 0.0131 |
| NaiveMean | 0.8175 ± 0.0422 |

### Experiment D — reliability heterogeneity (10 seeds)

Bayes is flat across reliability regimes — heterogeneity costs it
nothing — while the no-support baseline sits ~13 points lower throughout.
Means ± 95% CI computed from `results/expD_heterogeneity.csv`.

| Reliability | Bayes | MetaAttn | NoSupport |
|---|---|---|---|
| Homogeneous | 0.7279 ± 0.0083 | 0.6994 ± 0.0073 | 0.5965 ± 0.0054 |
| Moderate | 0.7267 ± 0.0106 | 0.7075 ± 0.0069 | 0.5890 ± 0.0065 |
| Heterogeneous | 0.7312 ± 0.0070 | 0.7059 ± 0.0058 | 0.5927 ± 0.0039 |

![Reliability heterogeneity](figures/fig4_reliability.png)

### Experiment H — correlated sources (10 seeds)

Breaking the conditional-independence assumption costs Bayes ~1.3 points
(0.9712 → 0.9582) and preserves the ranking over every baseline; naive
averaging stays near chance in both arms. Means ± 95% CI computed from
`results/expH_correlated.csv`.

| Arm | Bayes | ContextConditioned | SignedScalar | Naive |
|---|---|---|---|---|
| Independence | 0.9712 ± 0.0053 | 0.9623 ± 0.0056 | 0.8473 ± 0.0090 | 0.4490 ± 0.0132 |
| Correlated errors | 0.9582 ± 0.0059 | 0.9502 ± 0.0049 | 0.8435 ± 0.0144 | 0.4570 ± 0.0177 |

### Figures

All figures are in `figures/` as PNG and PDF; the `_n100` versions are
the 190-seed headline figures used in the paper.

![Representation ablations](figures/fig2_ablation_n100.png)

![Support-size scaling](figures/fig3_ksweep_n100.png)

![Multi-agent consensus](figures/fig5_consensus_n100.png)

![Stability boundary: population mapping vs per-source decoding](figures/figI_boundary_lines.png)

![Calibration](figures/fig_calibration_n100.png)

## Status

Manuscript complete (17 pages). Preprint posting (Zenodo) is planned, not
yet done. Target venue: AAMAS 2027 workshop, Hanoi, 3–4 May 2027;
recommended workshop paper deadline 24 January 2027. The AAMAS 2027 main
track closed 8 October 2026, so a workshop submission is the route. This
is a preprint citation, not a publication — the paper has not been
published at AAMAS 2026 or anywhere else.

## Reproduce

Run from the repository root so that `results/` and `figures/` are created
at the top level.

```bash
# One command: regenerate tables, figures and the paper PDF from the
# banked result CSVs — no experiment rerun needed (see REPRODUCIBILITY.md)
bash reproduce.sh

# Install dependencies
pip install -r requirements.txt

# Verify core components (~10 seconds)
python code/smoke_test.py

# Check the posterior-predictive fix against brute-force enumeration
python code/test_predictive_fix.py

# Rerun the experiments: the paper's headline tables use 190 seeds
python code/upgrade_n100.py --only A --seeds 100
python code/upgrade_n100.py --only BCE --seeds 30
# (10-seed subset only: python code/run_all_n10.py)

# Regenerate LaTeX table rows and figures from the CSVs
python code/gen_tables_n100.py
python code/make_figures_n100.py
```

Results are written to `results/*.csv`. Figures are written to
`figures/*.png` and `figures/*.pdf`. The compiled manuscript is at
`paper/main.pdf`; sources are in `paper/sections/`.

To reproduce the submitted tables from scratch, run the 190-seed suite
(`code/upgrade_n100.py --only A --seeds 100` and `--only BCE --seeds 30`;
takes hours, see REPRODUCIBILITY.md), then `code/gen_tables_n100.py` and
`code/make_figures_n100.py`. The 10-seed `code/run_all_n10.py` reproduces
a subset only.

Note: `data/` is intentionally not in the repository — raw datasets
(CIFAR-10H, ChaosNLI, CrowdTruth) are large and redistributed under their
own terms. The scripts that need external data document their source in
their docstrings, e.g. the CIFAR-10H CSV lookup in
`code/real_world_cifar10h.py`.

## Repository structure

```
.
├── code/
│   ├── pvi_experiments.py     # Full experimental suite (experiments A–G)
│   ├── upgrade_n100.py        # 190-seed headline runs (--only A / --only BCE)
│   ├── run_all_n10.py         # Run the full suite with N_SEEDS=10
│   ├── run_cg_n10.py          # Re-run experiments C–G only
│   ├── run_B_to_F.py          # Re-run experiments B–F only
│   ├── run_B_only.py          # Re-run experiment B only
│   ├── new_baselines.py       # EMDecoder and BayesSharedR baselines
│   ├── ctxlogreg_baseline.py  # CtxLogReg: per-source context-dependent logistic weights
│   ├── exp_correlated.py      # Experiment H: correlated source errors
│   ├── boundary_experiment.py # Experiment I: stability-boundary sweep
│   ├── real_world_cifar10h.py # PVI-REAL: CIFAR-10H real-data study (Dawid–Skene EM)
│   ├── pvi_real/              # PVI-REAL: controlled panels + CrowdTruth studies
│   ├── audit_fixes/           # October 2026 audit-repair experiments
│   ├── sens_bayes.py          # Grid-resolution and likelihood-temperature sensitivity
│   ├── ci_check.py            # 95% CI checks over the ten seeds
│   ├── cond_acc_analysis.py   # Supplementary: P(correct decision | convention recovery)
│   ├── gen_tables_n100.py     # LaTeX table rows from results CSVs (190-seed)
│   ├── make_figures_n100.py   # Figure generation, 190-seed tables
│   ├── smoke_test.py          # Standalone smoke test
│   └── test_predictive_fix.py # Brute-force test of the posterior-predictive log-odds
├── results/                   # Banked CSVs (reproduce.sh reads these)
├── figures/                   # PNG and PDF figures
├── paper/
│   ├── main.pdf               # Compiled manuscript (17 pages)
│   ├── main.tex               # LaTeX manuscript
│   ├── references.bib         # Bibliography
│   └── sections/              # Section files
├── notebooks/
│   └── pvi_colab.ipynb        # Colab runner
├── reproduce.sh               # One-command artefact regeneration
├── requirements.txt
├── environment.yml
├── REPRODUCIBILITY.md
└── README.md
```

## Conventions

The benchmark uses eight involutive conventions:

| ID | Convention | Context-dependent? |
|----|------------|:------------------:|
| 0 | identity | No |
| 1 | NOT | No |
| 2 | XOR with a1 | Yes |
| 3 | XOR with a2 | Yes |
| 4 | XOR with (a1 AND a2) | Yes |
| 5 | XOR with (a1 OR a2) | Yes |
| 6 | XOR with (a1 XOR a2) | Yes |
| 7 | XOR with NOT (a1 AND a2) | Yes |

All eight satisfy the involution property f_c(f_c(x, a), a) = x, which the
Bayesian aggregator relies on to recover the underlying reading from an
observed message.

## Reproducibility

- **Randomness:** all sampling uses Python's `random.Random` wrapped in a
  `SafeRNG` class. Each experiment seeds its own RNG via `SafeRNG(seed)`
  at the top of the experiment loop. PyTorch seeds are set via
  `torch.manual_seed(seed)` before every model initialisation.
- **Seeds:** per-experiment seeds 0 through 9 for the 10-seed runs;
  headline tables use 190 seeds (A: 100; B/C/E: 30 each).
- **Determinism of the Bayesian aggregator:** the grid over (ρ, r) is
  fixed and enumerated in full. No randomness.
- **Smoke tests:** `python code/smoke_test.py` verifies posterior
  normalization, posterior shape, finite log-odds, binary predictions,
  above-chance accuracy on a clean synthetic case, and RNG
  reproducibility.
- **Posterior-predictive fix:** `code/test_predictive_fix.py` checks the
  corrected posterior-predictive log-odds against explicit enumeration.
  See REPRODUCIBILITY.md for details of the correction.
- **CUDA nondeterminism:** not required for reproducibility. The suite
  produces the same numbers on CPU and GPU.

See REPRODUCIBILITY.md for the full checklist.

## Compute

- Reference environment: Google Colab T4 GPU
- Peak memory: < 2 GB
- Runtime (10 seeds): ~2 hours on T4, ~5–6 hours on CPU (observed
  September 2026)

## Citation

```bibtex
@misc{dubey2026provenance,
  title={Provenance-Aware Belief Aggregation in Multi-Agent Communication
         Under Convention Heterogeneity},
  author={Dubey, Jayant},
  year={2026},
  note={Preprint, manuscript in preparation for AAMAS 2027 workshop submission}
}
```

## License

MIT. See [LICENSE](LICENSE).

---

README revised November 2026; results as reported in October 2026, file
audit 21 November 2026.
