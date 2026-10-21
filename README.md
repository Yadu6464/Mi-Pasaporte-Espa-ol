# 2.4 GHz Microstrip Patch Antenna Optimization in HFSS

A modular, production-ready Python framework to design, simulate, and optimize an inset-fed microstrip patch antenna targeting **2.4 GHz** (ISM / Wi-Fi band).

The framework interfaces directly with **Ansys HFSS** via **PyAEDT** and provides two state-of-the-art derivative-free metaheuristics:
1. **Continuous Ant Colony Optimization ($\text{ACO}_{\mathbb{R}}$)** based on the Gaussian kernel formulation by Socha & Dorigo (2008).
2. **Covariance Matrix Adaptation Evolution Strategy (CMA-ES)** by Hansen et al., supporting both the official `cma` package and an integrated pure-NumPy engine.

Additionally, it incorporates a high-fidelity **physics-based analytical/surrogate simulator** (cavity resonance + transmission line model) allowing rapid testing, algorithm tuning, and benchmarking on machines without an active Ansys license.

---

## Architecture Overview

```
d:/projects/
├── configs/
│   ├── antenna_2_4ghz.yaml     # Antenna dimensions, materials, frequency specs
│   ├── aco_config.yaml         # Continuous ACO hyperparameters (k, m, q, xi)
│   ├── cmaes_config.yaml       # CMA-ES hyperparameters (sigma0, popsize)
│   └── hfss_config.yaml        # PyAEDT HFSS solver, passes, sweep settings
├── src/
│   └── antenna_opt/
│       ├── core/               # Configuration validation & result tracking
│       ├── models/             # Antenna parameters & analytical electromagnetics
│       ├── simulation/         # Base interface, PyAEDT HFSS runner, Mock runner
│       ├── optimizers/         # Base optimizer, ACO_R, CMA-ES
│       ├── objectives/         # S11 return loss, bandwidth & multi-criteria fitness
│       └── visualization/      # S11 response, convergence curves & dashboards
├── scripts/
│   ├── run_optimization.py     # Main CLI optimizer entry point
│   ├── benchmark_comparison.py # Side-by-side comparative benchmarking
│   └── export_hfss_model.py    # Standalone HFSS 3D CAD model generator
├── tests/                      # Comprehensive unit and integration test suite
├── pyproject.toml              # Project metadata & build setup
└── requirements.txt            # Python dependencies
```

---

## Antenna Design Parameters (2.4 GHz)

The inset-fed microstrip patch antenna is defined by five key parameters:

| Parameter | Symbol | Initial Analytical | Search Range | Physical Description |
| :--- | :---: | :---: | :---: | :--- |
| `patch_length` | $L$ | $29.4\text{ mm}$ | $[26.0, 34.0]\text{ mm}$ | Dominates fundamental resonant frequency $f_{10}$ |
| `patch_width` | $W$ | $38.0\text{ mm}$ | $[34.0, 44.0]\text{ mm}$ | Dictates radiation resistance & bandwidth |
| `inset_depth` | $y_0$ | $8.8\text{ mm}$ | $[5.0, 12.0]\text{ mm}$ | Tunes input impedance $R_{\text{in}}(y_0) \to 50\ \Omega$ |
| `inset_gap` | $g$ | $1.0\text{ mm}$ | $[0.5, 2.5]\text{ mm}$ | Isolates microstrip feedline notch |
| `feed_width` | $W_0$ | $3.05\text{ mm}$ | $[2.5, 3.8]\text{ mm}$ | $50\ \Omega$ microstrip transmission line width |

**Substrate Specifications**: Standard FR-4 ($\epsilon_r = 4.4, \tan\delta = 0.02, h = 1.6\text{ mm}, t = 35\ \mu\text{m}$). Configurable to Rogers RT/duroid 5880 in `configs/antenna_2_4ghz.yaml`.

---

## Quickstart

### 1. Installation

Install dependencies:
```bash
pip install -r requirements.txt
```

To install the package in editable mode:
```bash
pip install -e .
```

### 2. Running an Optimization

#### Offline Surrogate Mode (Instant, No HFSS Required):
```bash
# Run both Continuous ACO and CMA-ES:
python scripts/run_optimization.py --algorithm both --mode mock --max-evals 120

# Run only Continuous ACO (ACO_R):
python scripts/run_optimization.py --algorithm aco --mode mock --max-evals 100

# Run only CMA-ES:
python scripts/run_optimization.py --algorithm cmaes --mode mock --max-evals 100
```

#### Full-Wave Ansys HFSS Mode (PyAEDT):
```bash
# Connects to Ansys AEDT, creates 3D CAD model, excites port, and runs FEM solver
python scripts/run_optimization.py --algorithm cmaes --mode hfss --max-evals 50
```

### 3. Running the Comparative Benchmark Suite

To generate a complete side-by-side performance evaluation between $\text{ACO}_{\mathbb{R}}$ and CMA-ES:
```bash
python scripts/benchmark_comparison.py --mode mock --evals 120
```
This generates:
- `results/benchmark/benchmark_dashboard.png`: 4-panel figure comparing S11 response, convergence curves, performance metrics, and optimal dimensions.
- `results/benchmark/aco_benchmark.json` and `cma_benchmark.json`: Complete evaluation records and convergence trajectories.

### 4. Direct HFSS 3D Model Generation

To construct or inspect the parameterized 3D model directly in Ansys HFSS:
```bash
python scripts/export_hfss_model.py --patch-l 29.4 --patch-w 38.0 --inset-y0 8.8 --solve
```

---

## Optimization Algorithms

### 1. Continuous Ant Colony Optimization ($\text{ACO}_{\mathbb{R}}$)
Based on Socha and Dorigo (2008):
- Maintains a solution archive $T$ of size $k$, sorted by objective fitness ($f(s^1) \le \dots \le f(s^k)$).
- Assigns Gaussian kernel weights $w_l = \frac{1}{q k \sqrt{2\pi}} \exp\left(-\frac{(l-1)^2}{2 q^2 k^2}\right)$.
- Samples each coordinate from a normal distribution $\mathcal{N}(\mu_i^l, \sigma_i^l)$ with standard deviation proportional to the archive distance spread $\sigma_i^l = \xi \sum_{e=1}^k \frac{|s_i^e - s_i^l|}{k-1}$.
- Applies reflection boundary handling and archive replacement.

### 2. Covariance Matrix Adaptation Evolution Strategy (CMA-ES)
- Maintains an evolutionary search distribution $\mathcal{N}(m, \sigma^2 C)$.
- Dynamically adapts step-size $\sigma$ via cumulative path step-size control ($p_\sigma$).
- Adapts covariance matrix $C$ using rank-1 and rank-$\mu$ updates to capture parameter correlations (e.g., between patch length $L$ and inset depth $y_0$).
- Seamlessly falls back to a built-in pure-NumPy CMA-ES implementation if the C-extension `cma` library is unavailable.

---

## Verification & Testing

Run all unit and integration tests:
```bash
python -m unittest discover -s tests -p "test_*.py" -v
```
All tests verify:
- Microstrip transmission line and cavity model equations.
- Invariant boundary satisfaction and geometry feasibility checks.
- Convergence of Continuous ACO and CMA-ES on standard benchmarks.
- End-to-end optimization pipeline and automated result plotting.
