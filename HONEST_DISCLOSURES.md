# Honest Disclosures -- Glass PDK

> **Full transparency on limitations, assumptions, and validation boundaries**
> **Last updated:** February 2026

---

## Purpose

This document discloses every known limitation, simplification, and assumption in the Glass PDK platform. It is intended for technical auditors, potential licensees, and independent reviewers. We believe that honest disclosure of limitations strengthens rather than weakens confidence in the platform.

---

## 1. Fabrication Status: Computational Only

**No glass interposers have been fabricated.**

All results in this repository are from computational simulation. The physics solvers are validated against:
- Analytical closed-form solutions (coaxial cable impedance formula)
- Published experimental measurements (Sukumaran ECTC 2014, Watanabe ECTC 2015)
- Cross-solver benchmarking (BEM vs. 3D FDTD)

The platform is assessed at Technology Readiness Level (TRL) 4-6: validated in a laboratory simulation environment, not in a manufacturing environment. The gap between simulation and fabrication includes effects that are modeled approximately (surface roughness, via taper, plating voids) and effects that are not modeled at all (delamination at interfaces, contamination, lithography overlay errors).

**What this means for a potential licensee:** The platform produces design candidates that are physically sound in simulation. Fabrication validation is required before production deployment. We estimate 6-12 months of fab partnership to correlate simulation with silicon (glass) data.

---

## 2. BEM Solver Accuracy: 0.35% Context

The claimed 0.35% error is a verification of the 2D BEM solver against the analytical coaxial cable impedance formula:

```
Z0 = (60 / sqrt(epsilon_r)) * ln(b/a)
```

**What this validates:** The numerical method (boundary element discretization, Green's function integration, matrix inversion) is correctly implemented for ideal circular coaxial geometries.

**What this does not validate:**
- Accuracy for non-circular geometries (elliptical, rectangular, octagonal) -- these lack analytical solutions for direct comparison
- Accuracy in the presence of manufacturing artifacts (via taper, sidewall roughness, plating voids)
- Accuracy at frequencies above approximately 77 GHz where the quasi-static assumption breaks down and full-wave effects become significant
- Accuracy for closely-spaced via arrays where coupling effects dominate

The solver models via taper linearly using a profile function r(z) and applies the Hammerstad-Jensen roughness correction as a post-processing factor. These are standard industry approximations but have not been independently verified against fabricated glass TGV measurements.

---

## 3. Cost Model Methodology

### The 2-4x Range (Validated)

The cost comparison includes full process costs: TGV drilling (laser modification + wet etch), seed layer deposition (PVD), electroplating, CMP, patterning, and test. Pricing data is sourced from:

- Corning Eagle XG: published pricing and volume discount tiers
- Schott AF32 and Borofloat 33: published pricing
- AGC EN-A1: published pricing
- Fused Silica: published pricing
- Silicon CoWoS: industry analyst estimates (Yole Group, TechInsights)

The 2-4x advantage depends on:
- Volume tier (1, 100, 1K, or 10K wafers)
- Glass type (commodity borosilicate vs. specialty low-CTE)
- Process complexity (number of metal layers, via density)
- Panel vs. wafer processing (panel provides additional 2-3x throughput advantage)

### The 6.2x Figure (Substrate-Only)

The earlier "6.2x" claim compared bare substrate costs only ($800 glass vs. $5,000 silicon). This comparison is valid for substrate material cost but does not include processing costs. It is preserved in the reference data as a substrate-level metric but is not the primary cost claim.

### What the Cost Model Does Not Include

- Yield loss costs (assumed 90% in baseline; actual yields are process-dependent)
- Equipment capital costs (glass processing requires different equipment than silicon)
- Qualification and certification costs
- Supply chain risk premiums for a less mature technology

---

## 4. Monte Carlo Yield: Simulated Process Variation

The yield prediction assumes Gaussian distributions for all process variables:
- Via diameter: nominal +/- 1 um (1 sigma)
- Via pitch: nominal +/- 1 um (1 sigma)
- Glass dielectric constant: nominal +/- 2% (1 sigma)

**Limitations:**
- Actual manufacturing distributions may be non-Gaussian (skewed, bimodal, heavy-tailed)
- Systematic (non-random) variation components (across-wafer gradients, tool-to-tool differences) are not modeled
- The distributions are assumed based on published process capability data, not measured from a specific manufacturing line
- Correlation between process variables is assumed to be zero; in practice, variables may be correlated (e.g., via diameter and taper angle)

The Cpk values reported are predictions, not measurements. They indicate the expected process capability given the assumed distributions and are useful for comparing designs, but should not be quoted as manufacturing specifications without fab validation.

---

## 5. Thermal-Mechanical Coupling: Sequential, Not Fully Coupled

The multi-physics simulation is sequentially coupled:

1. Thermal solver calculates temperature distribution
2. Mechanical solver uses temperature to compute stress and deformation
3. Electrical solver uses temperature to update material properties and geometry

The coupling is one-directional. Specifically:
- Joule heating from the electrical solver is NOT fed back into the thermal solver
- Stress-induced geometry changes are NOT fed back into the electrical solver
- Temperature-dependent material nonlinearity is modeled as piecewise linear

**Impact assessment:** For most TGV designs, the self-heating is small (temperature rise < 10C for signal vias, potentially larger for power vias at high current). The sequential approximation introduces less than 1% error in impedance and less than 5% error in stress for typical signal via designs. For high-power PDN vias carrying 10A+ per via, the error may be larger and fully-coupled simulation would be recommended.

---

## 6. Crosstalk Floor: Theoretical vs. Practical

The array router reports a theoretical crosstalk floor of -100 dB for optimized GSSG patterns. This figure represents the inductive coupling limit calculated from the Neumann interaction integral in the far-field approximation.

**In practice, the achievable crosstalk floor is -60 to -80 dB** due to:
- Manufacturing defects (via position errors, metal thickness variation)
- RDL routing escape paths that create unintended coupling
- Substrate mode coupling at high frequencies
- Finite ground plane conductivity
- Package-level coupling through shared power distribution

The -100 dB figure is useful as a design target to demonstrate the capability of the optimization algorithm. It should not be quoted as an achievable specification for a manufactured product.

---

## 7. Paris Law: Initial Flaw Size Assumption

The fatigue crack propagation model (Paris Law: da/dN = C * (delta_K)^m) requires an initial flaw size a_0 to begin integration. The platform assumes a_0 = 1 um, which is a standard assumption for polished glass surfaces.

**Impact:** Fatigue life scales strongly with initial flaw size. If the actual initial flaw size is 10 um (possible for laser-drilled and etched surfaces), the predicted lifetime decreases by approximately one order of magnitude. The Coffin-Manson model (used for plastically-deforming materials like copper) does not depend on initial flaw size.

---

## 8. Material Database: Vendor Datasheets

All material properties (dielectric constant, CTE, fracture strength, resistivity, thermal conductivity) are sourced from official vendor datasheets (Schott, Corning, AGC) and published literature. These values represent typical or nominal specifications.

**Limitations:**
- Batch-to-batch variation in glass properties is not characterized
- Temperature-dependent property curves use linear approximations (constant temperature coefficients) rather than measured curves
- The "hypothetical" glass TitanGlass has been flagged and removed; all 14 glasses in the database are commercially available products
- Fracture strength values are for pristine glass; TGV processing (laser drilling, etching) may reduce the effective strength

---

## 9. ML Surrogate: Training Data Limitations

The physics-constrained surrogate model (R-squared = 0.9652) is trained on outputs from the BEM solver. The surrogate inherits all limitations of the BEM solver. Additionally:

- The training data covers a specific region of the design space (via diameter 20-100 um, pitch 50-500 um, frequency 1-77 GHz). Extrapolation outside this range is not validated.
- The physics-violation penalty enforces monotonicity but does not enforce all physical constraints (e.g., it does not enforce causality in S-parameters).
- The surrogate has not been validated against independent experimental data.

---

## 10. What This Platform Is and Is Not

**The Glass PDK IS:**
- A validated computational design tool for TGV interposer design
- A demonstration of coupled multi-physics simulation capabilities
- A generative engine for exploring patent-safe design architectures
- A framework for predicting manufacturing yield from process specifications

**The Glass PDK IS NOT:**
- A substitute for fabrication validation
- A certified manufacturing design tool (no ISO 9001 or ISO 26262 certification)
- A guarantee of patent freedom (the patent landscape analysis is computational, not legal advice)
- A commercially shipping product (it is a technology demonstration and IP portfolio)

---

**We believe that honest disclosure of these limitations demonstrates engineering integrity and provides a solid foundation for informed evaluation of the platform's capabilities.**
