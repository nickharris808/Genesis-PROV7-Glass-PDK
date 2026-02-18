# Glass PDK Solver Architecture Overview

> **Non-Confidential Technical Summary**
> No solver source code is included in this public repository.

---

## Architecture

The Glass PDK integrates 16+ physics solver modules into a single automated pipeline. The pipeline is orchestrated by the PDK compiler, which accepts a YAML process specification and invokes each solver in sequence to produce a validated design kit.

```
YAML Spec --> [Compiler] --> BEM --> Thermal --> Stress --> Fatigue
                                                             |
                      Design Kit <-- Rules <-- Patent <-- Yield
```

Each solver operates on a standardized geometry dictionary and returns structured results. The compiler collects all results, checks pass/fail criteria, and produces the final design kit including S-parameters, reliability reports, and design rules.

---

## Solver Inventory

### 1. Quasistatic BEM Solver

**Domain:** Electromagnetics
**Method:** 2D Boundary Element Method (Method of Moments)
**Purpose:** Extract per-unit-length R, L, G, C parameters and characteristic impedance Z0 for TGV geometries.

**Physics:**
- Solves the 2D Laplace equation in integral form using the free-space Green's function.
- Via perimeter discretized into N line-charge segments.
- Potential matrix filled, inverted to find charge density, integrated for capacitance.
- Inductance from TEM quasi-static relation: L = mu_0 * epsilon_0 / C_vacuum.
- Frequency-dependent skin effect and dielectric loss modeled via surface impedance.

**Validation:** 0.35% error against analytical coaxial cable formula. Cross-validated against published data from Sukumaran (ECTC 2014) and Watanabe (ECTC 2015).

### 2. Thermal Impedance Solver

**Domain:** Thermal-electrical coupling
**Method:** Temperature-dependent material property scaling
**Purpose:** Simulate impedance drift from -40C to +150C.

**Physics:**
- Dielectric drift: epsilon_r(T) = epsilon_r(T0) * [1 + alpha_epsilon * (T - T0)]
- Resistivity drift: rho(T) = rho(T0) * [1 + alpha_rho * (T - T0)]
- Geometric expansion: d(T) = d0 * [1 + CTE_metal * (T - T0)], p(T) = p0 * [1 + CTE_glass * (T - T0)]

**Key result:** Glass Z0 drift = 0.19% over 190C range; organic drift = 1.89% (10x worse).

### 3. Lame Stress Solver

**Domain:** Thermomechanics
**Method:** Analytical thick-wall cylinder (Lame equations)
**Purpose:** Compute radial and hoop stress at glass-metal interface.

**Physics:**
- sigma_radial = (E * delta_alpha * delta_T) / (2 * (1 - nu))
- Safety factor = glass_fracture_strength / sigma_radial
- Supports single-material and bi-metallic (liner + core) configurations.

**Key result:** Copper SF = 0.22 (failure). Bi-metallic SF = 5.56 (safe). Reduction = 25x.

### 4. Coffin-Manson Reliability Solver

**Domain:** Low-cycle fatigue
**Method:** Coffin-Manson equation
**Purpose:** Predict cycles-to-failure for plastically-deforming via fills.

**Physics:**
- Nf = (delta_epsilon_p / (2 * epsilon_f'))^(1/c)
- Applicable to copper and other ductile metals under thermal cycling.

### 5. Paris Law Crack Propagation

**Domain:** High-cycle fatigue / fracture mechanics
**Method:** Paris Law integration
**Purpose:** Predict fatigue life for elastic materials where crack growth governs.

**Physics:**
- da/dN = C * (delta_K)^m
- Integrated from initial flaw size a0 to critical crack length.
- Applicable to tungsten, molybdenum, and other elastic fills.

### 6. Monte Carlo Yield Simulator

**Domain:** Manufacturing yield prediction
**Method:** Latin Hypercube Sampling (LHS) with statistical process control
**Purpose:** Predict manufacturing yield before fabrication.

**Physics:**
- 10,000 LHS samples over Gaussian distributions of process variables.
- Each sample evaluated through BEM solver for impedance.
- Cpk = min((USL - mu)/(3*sigma), (mu - LSL)/(3*sigma))
- "Centered Probability" optimization shifts nominal design to maximize Cpk.

**Key result:** Optimized designs achieve 99.9997% (Six Sigma) predicted yield.

### 7. Array Router / Crosstalk Minimizer

**Domain:** Signal integrity
**Method:** Neumann interaction integral
**Purpose:** Optimize GSG/GSSG via patterns to minimize crosstalk.

**Physics:**
- M_ij = (mu_0 / 2*pi) * ln(1 + (2h/d_ij)^2) for mutual inductance.
- NEXT = (1/4)(K_C + K_L); FEXT = (1/2)(K_C - K_L)(v/l)
- Evaluates hexagonal, checkerboard, and genetic-algorithm patterns.

**Key result:** Optimized patterns achieve -100 dB theoretical crosstalk floor.

### 8. Power Delivery Network Analyzer

**Domain:** Power integrity
**Method:** Joule heating, Black's equation
**Purpose:** Size TGV arrays for high-current delivery.

**Physics:**
- R_DC = rho(T) * L / A
- Joule heating: P = I^2 * R
- Electromigration MTTF = A * J^(-n) * exp(E_a / (k*T))

### 9. Pad Transition Solver

**Domain:** Parasitic extraction
**Method:** Parallel-plate capacitance with fringe correction
**Purpose:** Extract landing pad parasitics and cascade with via model.

### 10. Novel IP Generator

**Domain:** Design automation
**Method:** Combinatorial exploration with constraint filtering
**Purpose:** Generate patent-safe TGV architectures.

**Pipeline:** 41,700 candidates -> physics filter -> patent filter -> TRL filter -> 317 Golden Designs.

### 11. 3D FDTD Solver (Benchmark)

**Domain:** Full-wave electromagnetics
**Method:** Yee grid FDTD (JAX-accelerated)
**Purpose:** Full-wave validation of BEM results.

**Physics:**
- Maxwell's curl equations on staggered grid.
- H^(n+1/2) = H^(n-1/2) - (dt/mu)(curl E^n)
- E^(n+1) = E^n + (dt/eps)(curl H^(n+1/2))

### Additional Solvers

- **Bi-metallic shell solver:** Specialized Lame calculation for multi-layer shells
- **Differential pair solver:** Odd/even mode impedance for differential signaling
- **Warpage simulator:** Substrate warpage under thermal loading
- **Electromigration lifetime:** Black's equation for current density limits
- **Measurement correlation:** Statistical correlation with experimental data
- **Frequency-aware co-optimizer:** Multi-objective optimization over frequency range

---

## Material Database

The solver suite draws from a validated material library:

- **14 commercial glasses:** Schott (Borofloat 33, AF 32, D 263 T, Xensation), Corning (Eagle XG, Willow, Lotus), AGC (EN-A1, Dragontrail), NEG (OA-10G), Fused Silica, Sapphire, SiC
- **7 conductive fills:** Copper, Tungsten, Molybdenum, GlidCop, Silver, Gold, Aluminium

All material properties sourced from vendor datasheets (Schott, Corning, AGC) and cross-verified against published literature.

---

## Manufacturing Process Awareness

The solvers model four key manufacturing steps and their artifacts:

1. **Laser modification and etch (LIDE):** Hourglass taper effect modeled as linear profile r(z).
2. **Seed layer deposition (PVD):** Multi-layer surface impedance model for Ti/Cu seed stack.
3. **Electroplating:** Void fraction parameter (0-1) modeling center voids and stress concentration.
4. **CMP:** Hammerstad-Jensen roughness correction for surface finish effects on conductor loss.

---

**Note:** This document describes the solver architecture at a conceptual level. No solver source code, implementation details, or proprietary algorithms are disclosed in this public repository.
