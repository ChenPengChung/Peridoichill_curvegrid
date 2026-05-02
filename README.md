# Periodic Hill Curvilinear Grid Generator

Steger-Sorenson Poisson + Vinokur tanh stretching grid tool for periodic hill DNS/LES simulations (GILBM solver).

---

## Features

| Mode | Description | Use Case |
|------|-------------|----------|
| **Mode 1** (Zeta-only) | Keep original Ni x Nj, adjust vertical stretching | Quick re-clustering on existing topology |
| **Mode 2** (Adaptive) | Freely set Ni x Nj, Poisson + P,Q control functions | Resolution change (coarsen/refine) |
| **Mode 3** (Variable gamma) | Keep Ni x Nj, compute gamma(y) from u_tau(y) | Post-CFD adaptive refinement for z+ < 1 |
| **--verify** | Post-simulation z+ check | Validate grid meets DNS wall-unit requirement |
| **--auto** | Non-interactive, reads `variables.h` | CI/batch integration with solver |

---

## Prerequisites

### Python Environment

```
Python >= 3.8
numpy
```

Optional (recommended):
```
scipy        # higher-order bicubic interpolation (Mode 2)
matplotlib   # grid visualization & comparison plots
```

Install:
```bash
pip install numpy scipy matplotlib
```

---

## Required Input Data

### 1. Reference Grid File (`.dat`, Tecplot format)

A 2D structured grid in Tecplot ASCII format. The file must contain:

```
TITLE = "..."
VARIABLES = "X" "Y"
ZONE T="...", I=<Ni>, J=<Nj>, F=POINT
<x1> <y1>
<x2> <y2>
...
```

- `I` = streamwise node count (NY)
- `J` = wall-normal node count (NZ)
- Data order: point-by-point, J-major (j varies slowest)

Example files included: `2.medium grid.dat`, `3.fine grid.dat`

### 2. For Mode 3 (Variable gamma) -- u_tau Data Files

Two spanwise-averaged wall friction velocity files from a prior CFD run:

- **Bottom wall** u_tau file (e.g., `28.Re5600_j257_zplus_top_spanavg_2nd.dat`)
- **Top wall** u_tau file (e.g., `29.Re5600_j257_zplus_bottom_normal_spanavg_2nd.dat`)

Format: Tecplot-style with columns `y`, `z`, `u_tau` (one row per streamwise station).

### 3. For `--auto` Mode -- `variables.h`

A C header file with `#define` macros:

```c
#define NY      257       // streamwise node count (= grid I)
#define NZ      129       // wall-normal node count (= grid J)
#define LZ      (3.036)   // channel height [h units]
#define LY      (9.0)     // streamwise length [h units]
#define GAMMA   2.0       // Vinokur stretching parameter
#define ALPHA   (0.5)     // clustering symmetry (0.5 = symmetric)
#define CFL     (0.5)     // (optional)
#define GRID_DAT_DIR  "path/to/grid/folder"
#define GRID_DAT_REF  "1.I257_J129_g2.0_a0.5.dat"
```

Required defines for `--auto`: `NY`, `NZ`, `ALPHA`, `GAMMA`, `GRID_DAT_REF`

### 4. For `--verify` Mode

```
grid_zeta_tool.py --verify <grid.dat> <bot_utau.dat> <top_utau.dat> <Re>
```

- The grid `.dat` must be the one used in the CFD run
- u_tau files must come from that same CFD run
- Re = Reynolds number (e.g., 5600, 10595)

---

## Usage

### Interactive Mode (default)

```bash
python grid_zeta_tool.py
```

Follow the prompts:
1. Select reference grid file
2. Choose mode (1/2/3)
3. Set parameters (GAMMA, ALPHA, Ni, Nj...)
4. Tool runs identity verification
5. Generates and exports new grid

### Auto Mode (non-interactive)

```bash
python grid_zeta_tool.py --auto
```

Reads all parameters from `variables.h` in the parent directory.

### Verify Mode (post-simulation)

```bash
python grid_zeta_tool.py --verify grid.dat bottom_utau.dat top_utau.dat 5600
```

---

## Key Parameters

| Parameter | Range | Description |
|-----------|-------|-------------|
| **GAMMA** | 0.0 -- 10.0 | Vinokur stretching intensity. 0=uniform, 2.0=recommended, >=5=extreme |
| **ALPHA** | 0.01 -- 0.99 | Clustering symmetry. 0.5=symmetric top/bottom |
| **Ni** | 10 -- 2000 | Streamwise nodes (Mode 2 only) |
| **Nj** | 10 -- 2000 | Wall-normal nodes (Mode 2 only) |
| **z+ target** | 0.1 -- 1.5 | DNS wall-unit target (Mode 3, recommend 0.9) |
| **Re** | > 1 | Reynolds number (Mode 3 & verify) |

---

## Output Files

| File | Description |
|------|-------------|
| `adaptive_*.dat` | New grid in Tecplot format (ready for solver) |
| `grid_data_*.txt` | Grid parameter summary report |
| `compare_*.png` | Side-by-side reference vs. new grid |
| `spacing_*.png` | Vertical spacing distribution plot |
| `gamma_field_*.dat` | Mode 3: gamma(y) Tecplot table |
| `gamma_field_*.png` | Mode 3: gamma/u_tau/z+ distribution plot |
| `sensitivity_*.dat/.png` | Mode 3: u_tau tolerance margin |
| `convergence_*.png` | Mode 2: Poisson solver convergence history |
| `zplus_verify_report.dat` | --verify: z+ check report |

---

## Directory Structure

```
Peridoichill_curvegrid/
├── grid_zeta_tool.py          # Main tool
├── 2.medium grid.dat          # Reference grid (medium)
├── 3.fine grid.dat            # Reference grid (fine)
├── Re5600/                    # Re=5600 case data & results
│   ├── 1.I257_J129_g2.0_a0.5.dat
│   ├── 12.j257_k129_g2.0_a0.5_grid_info.txt
│   ├── 28/29.*.dat           # u_tau spanwise-averaged data
│   └── 30.*.png              # z+ streamwise plot
├── Re10595/                   # Re=10595 case data & results
└── testing/                   # Development/test outputs
```

---

## Typical Workflow

```
1. Prepare reference grid (.dat, Tecplot format)
2. Choose GAMMA/ALPHA based on target z+ and Re
   ┌─────────────────────────────────────────────┐
   │  First run:  Mode 1 or 2, GAMMA=2.0        │
   │  After CFD:  Mode 3 with u_tau data         │
   │  Validate:   --verify with new u_tau        │
   └─────────────────────────────────────────────┘
3. Run tool → get new grid .dat
4. Copy .dat to solver input directory
5. Run CFD simulation
6. Extract u_tau → iterate with Mode 3 / --verify
```

---

## Notes

- Grid `.dat` files use Tecplot ASCII format (POINT packing)
- The tool auto-detects physical vs. code units and rescales accordingly
- GILBM stability checks are printed after grid generation
- For extreme GAMMA (>4), check the stretching ratio and GILBM stability warnings
