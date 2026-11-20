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

## Method

A finite-grid Bayesian latent-convention aggregator: each source's
reporting convention is treated as a latent variable and decoded jointly
with the underlying reading, instead of weighting messages directly. The
repository also contains structural limitation theorems (nonnegative-weight
insufficiency, scalar insufficiency under context-dependent conventions,
equal-reliability optimum, posterior consistency) and their proofs.

Ten experiments cover the convention-stability sweep (A), representation
ablations (B), support-size scaling (C), reliability heterogeneity (D),
multi-agent consensus (E), an LLM communication benchmark (F), a
real-human-judgment CIFAR-10H study, candidate-set misspecification,
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

## Status

Manuscript complete (17 pages). Preprint posting (Zenodo) is planned, not
yet done. Target venue: AAMAS 2027 workshop, Hanoi, 3–4 May 2027;
recommended workshop paper deadline 24 January 2027. The AAMAS 2027 main
track closed 8 October 2026, so a workshop submission is the route.

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
