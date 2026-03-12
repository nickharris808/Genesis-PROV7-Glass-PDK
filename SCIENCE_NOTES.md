# SCIENCE_NOTES.md -- PROV_7_GLASS_PDK

**Date:** 2026-02-28
**Triggered by:** Red-team audit (score: 4.0/10)
**Purpose:** Document all science fixes applied and remaining limitations

---

## Summary of Audit Findings and Fixes

A red-team audit identified four serious science flaws in the Glass PDK codebase.
This document records what was found, what was fixed, and what limitations remain.

---

## Issue 1: Crosstalk Off by ~100 dB (CRITICAL)

### What was wrong

The original `array_router.py` reported crosstalk values of -100 to -130 dB NEXT
for "Silence" patterns at 200 um pitch. Published HFSS simulations of coupled TGVs
at similar geometries (Sukumaran ECTC 2012, Watanabe T-CPMT 2016) show NEXT in the
range of -25 to -35 dB at 28 GHz. The original model was off by 60-100 dB.

### Root cause

The coupling model used bare-wire mutual inductance and mutual capacitance formulas
appropriate for infinite parallel wires in free space. It omitted:

1. **Substrate coupling** -- Displacement currents through the glass dielectric between
   vias dominate over direct wire-to-wire coupling above 10 GHz. This is the dominant
   coupling mechanism in glass interposers.

2. **Pad-to-pad coupling** -- Capture and escape pads at via top/bottom create
   capacitive coupling platelets that are not modeled. This contributes 3-10 dB of
   additional crosstalk at frequencies above 20 GHz.

3. **Radiation coupling** -- At mmWave frequencies (>40 GHz), electromagnetic
   radiation from via discontinuities creates coupling paths not captured by the
   quasi-static model.

4. **Surface wave coupling** -- Glass substrates can support surface waves that
   create long-range coupling between distant vias.

### What was fixed

1. **`crosstalk.py`**: Added an empirical ground-plane shielding correction factor
   that reduces the bare-wire coupling coefficients by a factor of 0.05-0.15,
   producing NEXT values in the -25 to -40 dB range at 200 um pitch. The factor is
   tuned to match published data from Sukumaran ECTC 2012.

2. **`array_router.py`**: Coupling floor clamped to -40 dB (realistic noise floor
   for glass TGVs) instead of the previous implausible values.

3. **`array_coupling.py`**: Default return value for no-aggressor case changed
   from -100 dB to -40 dB.

4. **README.md**: All crosstalk claims updated from -100 dB to "-30 to -40 dB"
   with explicit notes that FEM validation is required.

### Remaining limitations

- The shielding correction factor (0.05-0.15) is EMPIRICALLY FITTED to match
  published data, not derived from electromagnetic theory. This means the model's
  agreement with published measurements is circular: the correction was tuned TO
  match, so of course it matches.

- No independent validation against full-wave FEM (HFSS, CST, openEMS) has been
  performed.

- The model does not capture frequency-dependent coupling mechanisms (substrate
  modes, radiation), so extrapolation beyond the fitted range (10-40 GHz) is
  unreliable.

- For production design sign-off, full-wave 3D EM simulation is required.

### Files modified

- `glass_pdk/solvers/crosstalk.py` -- Added KNOWN LIMITATIONS docstring, empirical
  correction with science notes
- `glass_pdk/solvers/array_coupling.py` -- Fixed default floor from -100 to -40 dB
- `glass_pdk/solvers/array_router.py` -- Floor clamped to -40 dB
- `README.md` -- All crosstalk claims corrected (lines 129, 323, 1089-1090,
  2556-2557, 2912-2916)

---

## Issue 2: Circular BEM Validation (Tautology)

### What was wrong

The BEM solver validation (`bem_solver_validation.py`) compared the solver's output
against the analytical coaxial line formula Z0 = 60/sqrt(er) * ln(b/a). However,
this is the SAME formula that the solver implements at refinement level 1. Agreement
between the solver and the formula proves only that the code correctly implements
the formula -- not that the formula correctly predicts real TGV behavior.

The module was titled "BEM Solver Validation" and the README claimed it as
"Validated vs analytical" -- implying independent validation that did not exist.

### What was fixed

1. **`bem_solver_validation.py`**: Module docstring relabeled from "Validation" to
   "Internal Consistency Check." Added a comprehensive "VALIDATION GAPS" section
   listing 6 specific gaps (no FEM comparison, no fab measurements, no crosstalk
   validation, no loss validation, approximate published data ranges, unvalidated
   coaxial approximation).

2. **`due_diligence_simulation.py`**: Test 3 renamed from "Cross-Validation" to
   "Internal Consistency Check" with explicit notes that it compares the solver
   against its own formula.

3. **README.md**: Line 83 (audit table) updated from "Validated vs analytical"
   to "Internal consistency checks -- NOT independent FEM/measurement validation."
   Line 116 (S-TIER Verification) changed to "PARTIAL" with explanation.
   Line 1077 heading already corrected.

### What this module ACTUALLY does

1. **Internal consistency check**: BEM solver vs its own coaxial formula
   (confirms code correctness, not physical accuracy)

2. **Published range comparison**: Solver output compared to impedance ranges
   from paper abstracts (Sukumaran ECTC 2014, Watanabe ECTC 2015). These are
   approximate ranges, not digitized measurement data with error bars.

3. **Physics sanity checks**: Skin effect scaling, Z0 flatness, loss monotonicity.

### What is still needed for true validation

- Comparison against an independent 3D FEM solver (Ansys HFSS, CST Studio Suite,
  Palace, or openEMS) for 5+ representative geometries across 1-100 GHz
- Comparison against VNA S-parameter measurements of fabricated TGV test structures
- Validation of loss predictions against 4-port de-embedded measurements
- Validation of mutual coupling/crosstalk predictions against measured NEXT/FEXT

### Files modified

- `bem_solver_validation.py` -- Complete docstring rewrite, relabeled all functions
- `due_diligence_simulation.py` -- Test 3 relabeled
- `README.md` -- Lines 83, 116, 231

---

## Issue 3: "317 Golden Designs" Was a Grid Sweep

### What was wrong

The Novel IP Generator (`glass_pdk/solvers/novel_ip.py`) produces design points
by sweeping over combinations of:
- 7 fill metals x 6 glass types x multiple diameters x multiple pitches

All points that produced finite Z0 and stress values and met basic screening
criteria (safety factor > 1, Z0 between 40-60 Ohm, low prior art risk) were
labeled as "golden designs." The documentation claimed these were 317 (or 605)
individually optimized, validated "golden" designs worth "$1.5B+" -- in reality
they were parameter sweep points from a grid search.

### What was fixed

1. **Terminology**: All references to "golden designs" (when referring to the
   parameter sweep) renamed to "parameter sweep points" or "screened design
   points" throughout:
   - `glass_pdk/solvers/novel_ip.py` -- Variable renamed from `golden` to
     `screened`, print messages corrected
   - `design_kits.py` -- Variable renamed from `golden` to `passing`,
     docstrings updated
   - `patent_miner.py` -- Print message corrected
   - `README.md` -- Multiple sections updated (lines 39, 630-636, 1216,
     1603, 1687)
   - `DATA_ROOM.md` -- Already corrected in prior pass
   - `DESIGN_AROUND_DESERT.md` -- Already corrected in prior pass
   - `EXECUTION_PLAN.md` -- Corrected

2. **Screening criteria documented**: The actual screening criteria are now
   explicitly stated wherever the sweep points are referenced:
   - Safety factor > 1.0 (analytical Lame stress prediction)
   - Z0 between 40-60 Ohm (coaxial approximation)
   - Prior art risk rated as LOW or VERY LOW

3. **Disclaimers added**: Every reference to the screened points now includes
   a note that these are analytically screened, not individually validated,
   and each requires independent FEM/measurement validation before production use.

### What the screening criteria actually mean

- **Safety factor > 1.0**: The analytical Lame thick-wall cylinder stress model
  predicts that the peak glass stress is below the glass fracture strength. This
  is a necessary but not sufficient condition for reliability -- it does not
  account for fatigue, stress concentrators at via corners, or process-induced
  defects.

- **Z0 between 40-60 Ohm**: The coaxial approximation Z0 = 60/sqrt(er)*ln(b/a)
  falls near 50 Ohm. This is an idealized model that does not account for pad
  loading, via taper, or surface roughness.

- **Low prior art risk**: Based on patent landscape analysis of Intel, TSMC, and
  Samsung portfolios. This is a legal analysis, not a physics validation.

### What would make a design truly "golden"

A design should only be labeled "golden" or "qualified" if it meets ALL of:
1. Independent FEM simulation (HFSS/CST) confirms Z0 and S-parameters
2. Fabricated test structures measured with VNA confirm FEM predictions
3. Thermal cycling reliability tested to target cycle count
4. Electromigration lifetime validated at operating current/temperature
5. Manufacturing yield verified on pilot line

### Files modified

- `glass_pdk/solvers/novel_ip.py`
- `design_kits.py`
- `patent_miner.py`
- `README.md`
- `EXECUTION_PLAN.md`

---

## Issue 4: TGV Design Rules

### What was checked

The PDK design rules were verified against published glass vendor capabilities:

| Parameter | PDK Rule | Corning | AGC (LIDE) | Schott | Assessment |
|-----------|----------|---------|------------|--------|------------|
| Min diameter | None specified | 25 um | 20 um | 20 um | OK (sweep starts at 20 um) |
| Max aspect ratio | AR <= 10:1 | 8:1 typical | 10:1 | 8:1 | AGGRESSIVE for Corning/Schott |
| Min pitch clearance | diameter + 20 um | diameter + 50 um | diameter + 40 um | diameter + 50 um | TOO AGGRESSIVE |
| Min glass web | Not specified | 25-50 um | 25 um | 25-50 um | MISSING from PDK |

### What was already in place

The `report.py` template already includes a detailed IMPORTANT section noting that
the 20 um pitch clearance rule is aggressive and may not be achievable with current
glass drilling technology. The `design_kits.py` already includes a `glass_vendor_constraints`
section with Corning, AGC, and Schott data.

### Remaining gaps

1. The **20 um pitch clearance** used in the PDK is 2-2.5x more aggressive than
   what glass vendors specify. Real glass foundries typically require diameter + 40-50 um
   minimum pitch to ensure adequate glass web strength.

2. The **AR <= 10:1** rule matches AGC's LIDE process capability but exceeds
   Corning and Schott's typical 8:1 limit.

3. The PDK does not enforce a **minimum glass web thickness** rule. This is a
   critical manufacturing constraint (25-50 um minimum from all vendors).

4. The PDK does not account for **drilling tolerance** (typically +/- 2-5 um on
   diameter and +/- 5-10 um on position), which affects minimum pitch.

### Recommendations

- Add minimum glass web rule: 25 um minimum between any two via edges
- Change default pitch clearance to diameter + 50 um for conservative designs
- Add drilling tolerance as a design rule parameter
- Document that AR > 8:1 requires AGC LIDE process specifically

---

## Overall Assessment After Fixes

### What is now correctly represented

1. **Crosstalk**: Values are in the -25 to -40 dB range, consistent with published
   measurements. Explicit notes state this is empirically fitted and requires FEM
   validation.

2. **BEM validation**: Clearly labeled as internal consistency check, not independent
   validation. Validation gaps explicitly listed.

3. **Design database**: Parameter sweep points are honestly labeled as screening
   results, not validated designs. Screening criteria are documented.

4. **Design rules**: Analytical constraints are distinguished from foundry process
   constraints. Real vendor data is included for comparison.

### What remains unvalidated

1. No independent FEM (HFSS/CST) cross-validation of any solver predictions
2. No comparison against fabricated device measurements
3. Crosstalk model relies on empirical fitting, not physics-based correction
4. Reliability predictions (Lame, Coffin-Manson, Paris Law) are analytical only,
   not calibrated against experimental fatigue data
5. Cost model uses approximate industry pricing, not foundry quotes
6. Design rules are more aggressive than vendor specifications

### Recommended science score improvement path

| Action | Estimated Score Impact |
|--------|----------------------|
| FEM cross-validation (5 geometries, 1-100 GHz) | +1.5 |
| Fabricated test structure measurements | +2.0 |
| Calibrate crosstalk model against HFSS | +1.0 |
| Validate reliability against thermal cycling data | +1.0 |
| Add minimum glass web design rule | +0.5 |

With all of the above completed, the science score could reasonably reach 8-9/10.

---

## References

1. Sukumaran et al., "Through-Package-Via Formation and Metallization of Glass
   Interposers," ECTC 2014. (TGV impedance measurements)
2. Watanabe et al., "Development of Low-Cost Glass Interposer with Through-Glass
   Vias," ECTC 2015. (TGV impedance and loss data)
3. Sukumaran et al., "Low-Cost Thin Glass Interposers as a Superior Alternative
   to Silicon and Organic Interposers," ECTC 2012. (Crosstalk measurements)
4. Pozar, "Microwave Engineering," 4th ed. (Coaxial line theory)
5. Harrington, "Field Computation by Moment Methods," 1968. (BEM/MoM reference)
6. Jackson, "Classical Electrodynamics," 3rd ed. (Two-cylinder capacitance)
7. Corning Glass Technologies, "Eagle XG Glass Substrate Design Guide," 2024.
8. AGC Inc., "AN-Wizus TGV Design Rules," 2024.
9. Schott AG, "Borofloat 33 Datasheet," 2024.
