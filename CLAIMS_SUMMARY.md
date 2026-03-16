# Glass PDK Patent Claims Summary

> **Non-Confidential Summary | Full claim text is not disclosed**
> **Total: 75 claims across 8 provisional patent applications + IP library**

---

## Overview

The Glass PDK intellectual property portfolio comprises 8 provisional patent applications and an IP library, totaling 75 claims. The claims cover the complete design-to-manufacture flow for glass substrate interposers: from automated PDK compilation and CTE co-optimization through ML-accelerated design exploration to manufacturing yield prediction.

No full claim text is reproduced in this public repository. The summaries below describe the scope and subject matter of each patent family.

---

## Patent Family 1: Comprehensive Glass Substrate PDK

**Title:** System and Method for Automated Design, Optimization, and Verification of Through-Glass Via Interconnects Using Multi-Physics Simulation

**Claims:** 10

**Scope:** A computer-implemented method for designing Through-Glass Via interconnects comprising simultaneous evaluation of impedance matching, thermomechanical stress, and fatigue life constraints to identify optimal material-geometry combinations. The system compiles YAML specifications into validated design kits using coupled physics solvers. Covers the end-to-end pipeline from specification ingestion through solver orchestration to design kit output.

**Key innovation:** No prior system integrates BEM impedance extraction, Lame stress analysis, Coffin-Manson reliability, and Monte Carlo yield prediction into a single automated pipeline for glass substrates.

---

## Patent Family 2: CTE Co-Optimization

**Title:** Method for Increasing Thermomechanical Reliability of Glass Interposers using Bi-Metallic and CTE-Matched Via Fill Materials

**Claims:** 5

**Scope:** A method for co-optimizing the coefficient of thermal expansion (CTE) match between via fill materials and glass substrates across the full temperature range (-40C to +150C). Includes the Bi-Metallic Shell architecture where a CTE-matched liner (tungsten or molybdenum) is placed between the glass and a conductive core (copper), reducing radial stress at the interface below the glass fracture strength.

**Key innovation:** CTE matching within 2.0 ppm/K reduces thermomechanical stress by 25x, converting glass interposers from fragile structures to designs that survive billions of thermal cycles.

---

## Patent Family 3: Pad Transition Design

**Title:** Method for Extracting and De-Embedding Parasitic Impedance of Landing Pads and Redistribution Layers in Glass Interposers

**Claims:** 6

**Scope:** A method for extracting parasitic parameters of TGV interconnects comprising parallel-plate capacitance computation of landing pads with fringe field correction, and cascading said capacitance with the via body model to determine total insertion loss. Covers broadband pad transition optimization for impedance matching at mmWave frequencies.

**Key innovation:** De-embedding the pad transition parasitics enables accurate system-level signal integrity prediction for glass interposers, which was previously only possible with expensive 3D full-wave simulation.

---

## Patent Family 4: ML Surrogate for Design

**Title:** Method for Training Physics-Constrained Surrogate Models for Semiconductor Interconnects Using Hybrid Loss Functions

**Claims:** 5

**Scope:** A method for training a neural network surrogate model where the loss function includes a physics-violation penalty term that enforces monotonicity of fatigue life with respect to stress. The surrogate replaces full BEM simulation for rapid design space exploration while ensuring physical consistency in untrained regions of the parameter space.

**Key innovation:** The ML surrogate achieves R-squared = 0.9652, enabling 1000x faster design iteration than direct solver evaluation. The "physics constraint" is a non-negativity penalty (ReLU on negative outputs) -- not a true monotonicity enforcement via gradient penalties as described in the patent claims. The patent describes gradient-based monotonicity constraints, but the implemented code uses only output non-negativity.

---

## Patent Family 5: Monte Carlo Yield Prediction

**Title:** Method for Predicting Manufacturing Yield of Through-Glass Vias Using Latin Hypercube Sampling of Process Tolerances

**Claims:** 5

**Scope:** A computer-implemented method for predicting manufacturing yield comprising sampling process variables (via diameter, pitch, dielectric constant) using Latin Hypercube Sampling, simulating electrical performance of each sample through the physics pipeline, and computing the Cpk process capability index. Includes a "Centered Probability" optimization that shifts the nominal design to maximize yield.

**Key innovation:** Predicts Six Sigma (99.9997%) yield before fabrication by systematically exploring the multi-dimensional process variation space.

---

## Patent Family 6: Feasibility Analysis Framework

**Title:** System and Method for Automated Generation of Technical Feasibility Dossiers for Advanced Packaging Technologies

**Claims:** 5

**Scope:** A system for automated feasibility verification comprising a requirement ingestion module, a physics evaluation kernel that determines the optimal design point, and a document compiler that generates a compliance matrix and risk assessment. The system produces complete feasibility reports from a single specification input.

**Key innovation:** Automated feasibility reporting reduces the design evaluation cycle from weeks (manual multi-tool analysis) to minutes (single automated pipeline).

---

## Patent Family 7: Design Rules for Manufacturing

**Title:** Method for Extracting Geometric Design Rules from Multi-Physics Feasibility Boundaries

**Claims:** 5

**Scope:** A method for generating design rules by identifying the boundary of the feasible design space where reliability or electrical constraints are violated, and encoding said boundary as geometric rules (minimum pitch-to-diameter ratio, maximum via density, minimum spacing). The rules are automatically extracted from the physics simulation results rather than manually specified.

**Key innovation:** Physics-derived design rules replace empirical rules-of-thumb, ensuring that every rule has a traceable physical basis and quantified margin.

---

## Patent Family 8: Thermal-Z Stack Optimization

**Title:** Method for Simulating Temperature-Dependent Characteristic Impedance of Glass Interconnects

**Claims:** 6

**Scope:** A method for simulating signal integrity comprising updating the dielectric constant and physical dimensions of the interconnect based on temperature and CTE, and re-computing characteristic impedance at a plurality of temperature points. Covers Z-direction thermal management for glass interposers and through-glass thermal via optimization.

**Key innovation:** Quantifies the 10x thermal stability advantage of glass over organic substrates, enabling calibration-free operation in automotive (-40C to +150C) environments.

---

## Glass PDK IP Library (Supplementary Claims)

**Title:** Glass PDK IP Library -- Additional Provisional Claims

**Claims:** 28

**Scope:** Additional claims covering manufacturing-specific methods for TGV fabrication, process-specific patent claims for glass interposer production sequences, and glass composition optimization methods. These claims extend the core 8 patent families to cover the full manufacturing workflow from glass substrate selection through via formation to final test.

---

## Summary Table

| Family | Title (Short) | Claims |
|---|---|---|
| 1 | Comprehensive Glass Substrate PDK | 10 |
| 2 | CTE Co-Optimization / Bi-Metallic Shell | 5 |
| 3 | Pad Transition Parasitic Extraction | 6 |
| 4 | ML Surrogate with Physics Constraints | 5 |
| 5 | Monte Carlo Yield Prediction | 5 |
| 6 | Automated Feasibility Reporting | 5 |
| 7 | Inverse Design Rule Extraction | 5 |
| 8 | Thermal-Z Stack Optimization | 6 |
| Lib | Glass PDK IP Library | 28 |
| **Total** | | **75** |

---

---

## Important Disclosure: BEM Solver Impedance Accuracy

The BEM solver uses a coaxial approximation (`Z0 = (60/sqrt(er)) * ln(b/a)`) that produces characteristic impedance values in the range of **9-18 Ohm** for typical TGV geometries targeting 50 Ohm. This is an inherent property of the coaxial model applied to realistic TGV pitch-to-diameter ratios and high-Dk glass substrates — the small `ln(b/a)` ratio drives impedance well below 50 Ohm.

The claimed 0.35% solver accuracy is verified only against this same coaxial analytical formula (solver vs. its own reference formula), which is a circular validation, not an independent experimental check. The "50 Ohm" labels that appear in golden design kits (e.g., `build_golden_50ohm/`) reflect the target specification, **not the actual computed impedance**.

For full details on solver limitations and validation boundaries, see `HONEST_DISCLOSURES.md` (Section 2: BEM Solver Accuracy and Section 6: Crosstalk Floor, which includes the impedance mismatch disclosure).

---

**Note:** This summary is provided for informational purposes only. Full claim text, dependent claim hierarchies, and prosecution details are maintained in the confidential data room (PROV_7_GLASS_PDK). No patent claim text is reproduced in this public repository.
