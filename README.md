# SolKin — Mode-Resolved Kinetic Transport Solver

SolKin is independent research software for deterministic simulations of
two-dimensional, massless kinetic transport. It evolves spatial and angular
distributions from anisotropic initial conditions, extracts harmonic flow
responses, and compares separable relaxation models with a momentum-resolved
Fokker–Planck collision kernel.

The repository provides two compatible solver engines:

- `mrta.solver.Solver` evolves the momentum-integrated distribution
  `Phi(x, y, phi, tau)` with selectable mode-dependent relaxation spectra.
- `mrta.solver_p.SolverP` evolves the momentum-resolved distribution
  `F(x, y, p, phi, tau)` with either the same scalar spectra or a non-separable
  Fokker–Planck operator.

Both engines use exact spectral advection, Landau matching, Strang operator
splitting, and exact collision exponentials. They support full evolution,
one-hit response calculations, centered-response extraction, Bjorken dilution,
and reproducible JSON output.

## Main capabilities

- Harmonic responses `kappa_n = d v_n / d epsilon_n` for `n = 2, 3, 4, ...`.
- Scalar relaxation spectra: `flat`, `diff`, `mcdiff`, and `mixed`.
- Momentum-resolved leading-log Fokker–Planck/Landau kernel.
- Full collision evolution and dilute one-hit accumulation.
- Positive-definite default initial condition:

  ```text
  E0(r,theta) proportional to exp[-r^2/(2 R^2)]
                         [1 + beta/2 (r/R)^n cos(n theta)]^2
  ```

- Legacy additive initial condition through `ic="additive"` for analytic
  validation oracles.
- Fixed-coupling, conformal-rate, local-temperature, infrared-cutoff,
  matched-opacity, and momentum-bin studies.
- Resumable opacity scans and convergence scripts.
- Doxygen-compatible source documentation.

## Installation

Python 3.10 or newer is recommended.

```bash
python -m pip install numpy scipy numba matplotlib pytest
```

The standalone validation commands do not require pytest itself, but the
numerical dependencies above are required.

## Quick start

Run the reduced validation suites:

```bash
python tests/test_validation.py --fast
python tests/test_fp.py --fast
```

Run one momentum-integrated scan point:

```bash
python run_scan.py \
    --spectrum diff \
    --g 0.1 \
    --harmonics 2 3 4 \
    --Nx 64 \
    --Nphi 64 \
    --tau_max 4.0 \
    --dt 0.02 \
    --out results/example
```

Run one momentum-resolved Fokker–Planck point:

```bash
python run_scan_fp.py \
    --kernel fp \
    --g 2.0 \
    --harmonics 2 3 \
    --out results_fp
```

Argument spelling is driver-specific. For example, `run_scan.py` uses
`--tau_max` but `--one-hit`. Check the exact interface with
`python DRIVER.py --help`.

## Solver engines

### Momentum-integrated MRTA solver

`mrta/solver.py` evolves `Phi(x, y, phi, tau)` according to a local relaxation
rate whose angular dependence is specified by `gamma_hat(l)`. The available
spectra are:

| Spectrum | Purpose |
|---|---|
| `flat` | Common rate for all relaxing harmonics. |
| `diff` | Diffusion-like harmonic hierarchy. |
| `mcdiff` | Modified diffusion hierarchy. |
| `mixed` | Tunable combination controlled by `b_over_a`. |

Advection is applied as an exact Fourier-space translation. The collision
substep uses a closed-form Landau-matched equilibrium and exact exponential
relaxation for every harmonic.

### Momentum-resolved FP solver

`mrta/solver_p.py` evolves `F(x, y, p, phi, tau)`. For `kernel="fp"`, each
angular harmonic is acted on by a radial momentum-space operator from
`mrta/fp_kernel.py`. The implementation includes:

- conservative Sturm–Liouville discretization;
- exact eigenfactorization of collision exponentials;
- optional infrared cutoff and inner boundary condition;
- conservation-projected gain sector;
- optional conformal local-temperature resampling;
- momentum-binned harmonic observables;
- separate direct and equilibrium-subtraction one-hit contributions.

Using a scalar kernel such as `kernel="flat"` sends `SolverP` through the same
momentum-resolved pipeline and provides a direct equivalence check against
`Solver`.

## Validation gates

The permanent validation tests live in `tests/`. Gate numbering is retained
for compatibility with earlier result logs. V5 and V6 are not omitted files:
linearity and grid/time/box convergence are study-specific scripts rather than
permanent unit gates. V7 is the permanent initial-condition symmetry gate.

### Integrated-solver gates

Implemented in `tests/test_validation.py`:

| Gate | Check | Acceptance criterion |
|---|---|---|
| V1 | Spectral advection against exact free streaming using the additive oracle family. | Maximum relative error `< 1e-8`. |
| V2 | Homogeneous decay of the `l = 2` and `l = 3` modes for `flat`, `diff`, and `mcdiff`. | Relative error `< 1e-10`. |
| V3 | Energy and momentum conservation in a coupled eccentric run. | `|Delta E/E| < 1e-12` and `max|P|/E < 1e-12`. |
| V4a | Full evolution approaches the one-hit response in the dilute limit. | Relative difference `< 5%`. |
| V4b | Dilute double-ratio theorem for `D32`. | One-hit agreement at numerical precision; full result within its finite-opacity tolerance. |
| V7 | Antisymmetry under `beta -> -beta` for the positive-definite initial condition. | Machine-scale odd-harmonic residual and even-harmonic relative residual `< 1e-4`. |

### Momentum-resolved FP gates

Implemented in `tests/test_fp.py`:

| Gate | Check | Acceptance criterion |
|---|---|---|
| V0p | `SolverP(kernel="flat")` reproduces the integrated flat-spectrum solver. | Maximum harmonic difference `< 5e-4`. |
| V1p | Numerical Rydberg spectrum and exact collision-substep eigenmode decay. | Eigenvalue error `< 5e-4`; substep error `< 1e-10`. |
| V2p | Exact energy and momentum conservation in a coupled momentum-resolved run. | Energy and momentum residuals `< 1e-12`. |
| V3p | Full FP evolution approaches its one-hit accumulator as `g -> 0`. | Relative difference `< 2%`. |
| V4p | Direct dilute anchor and rotation covariance. | Anchor error `< 5e-3`; symmetry residual `< 1e-8`. |

Run all gates directly:

```bash
python tests/test_validation.py --fast
python tests/test_fp.py --fast
```

Run the full-resolution gates by omitting `--fast`:

```bash
python tests/test_validation.py
python tests/test_fp.py
```

With pytest installed:

```bash
python -m pytest tests --fast
```

### Current fast-suite results

The merged code was checked on 2026-07-22 with:

```bash
python tests/test_validation.py --fast
python tests/test_fp.py --fast
```

All permanent gates passed. The measured numerical residuals were much smaller
than their acceptance thresholds:

| Gate | Measured result |
|---|---|
| V1 | Maximum free-streaming relative error `4.14e-11` (threshold `1e-8`). |
| V2 | Homogeneous `l = 2, 3` decay agrees with the analytic exponential for all three scalar spectra at the reported numerical precision. |
| V3 | `|Delta E/E| = 2.96e-14`; `max|P|/E = 3.65e-16`. |
| V4a | Full and one-hit dilute responses agree to the displayed precision (`0.0%` relative deviation). |
| V4b | The measured dilute double ratio is `D32 = 2.2500`, matching the analytic value `9/4`. |
| V7 | Odd-harmonic absolute residuals are at most `6.92e-15`; even-harmonic relative residuals are `4.6e-6` to `5.7e-6`. |
| V0p | Maximum scalar-pipeline harmonic difference `6.74e-5` (threshold `5e-4`). |
| V1p | Rydberg eigenvalue errors `2.2e-4` to `2.5e-4`; collision-substep error `2.7e-15`. |
| V2p | `|Delta E/E| = 2.6e-15`; maximum momentum residual `3.5e-15`. |
| V3p | Full/one-hit relative difference `0.52%` at `g = 0.02` (threshold `2%`). |
| V4p | Direct `D32 = 2.2439`, equal to the discrete prediction within `2.6e-7`; rotation residual `9.8e-14`. |

These numbers certify the reduced-grid smoke configuration. Production claims
should additionally use the dedicated linearity, time-step, spatial-grid, and
finite-box convergence scripts listed below.

## Numerical validation scripts

The `scripts/` directory contains reproducible production and convergence
drivers. Important examples include:

| Script | Purpose |
|---|---|
| `01_full_scan.sh` | Standard integrated full-evolution scan. |
| `02_linearity_g8.sh` | Centered-response linearity check. |
| `03_onehit_scan.sh` | Integrated one-hit scan. |
| `04_convergence_nx.sh` | Spatial-grid convergence. |
| `05_convergence_dt.sh` | Time-step convergence. |
| `06_convergence_box.sh` | Finite-box convergence. |
| `08_tau0_fixed_g.sh` | Fixed-coupling initialization-time scan. |
| `09_tau0_matched_opacity.sh` | Matched-opacity initialization-time scan. |
| `12_extended_g_scan.sh` | Extended integrated-opacity scan. |
| `13_extended_g_validation.sh` | Extended-opacity endpoint validation. |
| `15_asymptotic_g_scan.sh` | Large-opacity asymptotic scan. |
| `17_fp_d32_high_opacity.sh` | Higher-opacity FP and matched-flat `D32` scan. |

Scripts are resumable where practical and skip completed JSON records.

## Production and analysis drivers

| File | Purpose |
|---|---|
| `run_scan.py` | General momentum-integrated scan driver. |
| `run_scan_fp.py` | Matched scalar/FP point driver. |
| `run_local_suite.py` | Resumable local FP baseline and robustness suite. |
| `run_ir_robustness.py` | Infrared cutoff, momentum grid, and boundary-condition study. |
| `run_chi2_matched.py` | Matched-opacity checks. |
| `run_alpha13.py` | Conformal-rate scan. |
| `run_localT.py` | Local-temperature study. |
| `run_gainproj.py` | Conservation-projection comparison. |
| `run_pbins.py` | Momentum-binned harmonic response. |
| `analyze_fp.py` | FP tables and benchmark plots. |
| `analyze_extended.py` | Extended-opacity summary. |
| `plot_d32_extended.py` | Integrated-solver `D32` plot and CSV export. |

## Repository structure

```text
mrta/
├── mrta/
│   ├── analytic.py       # closed-form validation oracles
│   ├── fp_kernel.py      # radial Fokker–Planck operator
│   ├── solver.py         # momentum-integrated solver
│   └── solver_p.py       # momentum-resolved solver
├── tests/                # permanent V-gate suites
├── scripts/              # reproducible scans and convergence checks
├── results_fp/           # supplied FP reference records and figures
├── run_scan.py
└── run_scan_fp.py
```

Generated results should normally be placed in a dedicated subdirectory under
`results/` or `results_fp/`. Do not merge convergence records with production
records when plotting: they may share `(kernel, g, harmonic)` keys while using
different numerical resolutions.

## Output format

Scan drivers write one JSON record per kernel or spectrum and opacity. Records
contain the physical parameters, numerical grid, response coefficients,
eccentricities, and any requested validation diagnostics. Example:

```json
{
  "kernel": "fp",
  "g": 2.0,
  "Nx": 80,
  "Nphi": 32,
  "Np": 20,
  "kappa": {
    "2": 0.0032,
    "3": 0.0007
  }
}
```

## Doxygen documentation

All core Python sources use Doxygen-style docstrings and comments. Generate the
API documentation with:

```bash
doxygen Doxyfile
```

The HTML output is written to `docs/html/`.

## Numerical guidance

- Always record `Nx`, `Nphi`, `Np`, `pmax`, `L`, `tau0`, `tau_max`, and `dt`.
- Check centered-response linearity at the largest opacity used in a result.
- Reduce `dt` as opacity grows even though collision exponentials are exact;
  the advection–collision splitting error can still grow.
- Validate important endpoints with at least one finer time step and grid.
- Use matched flat-kernel points when constructing FP double ratios.
- Keep convergence outputs separate from production scan directories.

## Theory references

- P. B. Arnold, G. D. Moore, and L. G. Yaffe, JHEP 11 (2000) 001,
  arXiv:hep-ph/0010177.
- J. Hong and D. Teaney, Phys. Rev. C 82 (2010) 044908,
  arXiv:1003.0699.
- H. Risken, *The Fokker–Planck Equation*, 2nd edition, Springer (1989).


## Requirements

numpy, numba (tested: numpy 2.4, numba 0.66). Single scan point at production
resolution (Nx = 192, Nphi = 128, dt = 0.02, tau_max = 8): a few minutes on
one node; the full money-plot scan (4 spectra x 12 opacities x 3 harmonics)
is a parallel job array.
##  Citation
If you use MRTA in research leading to a publication, please cite both:

1. The associated scientific paper:
   O. Soloveva et al., “Exact Factorization of the Eccentricity-to-Flow Response in Kinetic Theory”, ARXIV, 2607.xxxx.

2. The specific MRTA software release:
   O. Soloveva, MRTA: 2D mode-resolved transport solver,
   version X.Y.Z, Zenodo DOI.

Please also report the software version or Git commit used.
