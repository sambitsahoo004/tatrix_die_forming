# Blank-Holder-Free Deep Drawing Through a Tapered (Conical) Die

Finite element simulation of the deep drawing of an aluminium-killed steel sheet through a
30° conical die without a blank holder, built and solved in Abaqus/Standard 2021.

The model captures the full nonlinear forming process: large plastic strain, planar
anisotropy of the rolled sheet, and frictional contact between a deformable blank and two
analytical rigid tools.

---

## 1. Process

In conventional deep drawing a blank holder presses the flange against the die face to
suppress wrinkling. A tapered die replaces that function geometrically: the conical entry
supports the flange as it is drawn radially inward, so the tooling collapses from three
active surfaces to two. This removes the blank-holder force as a process variable, but it
raises the risk of both wrinkling (if the cone is too shallow) and fracture at the punch
nose (if drawing resistance is too high).

The model here is a single-stage draw at a **drawing ratio of 2.5**, which is at the
aggressive end for a low-carbon steel and is the reason the case is worth simulating rather
than estimating analytically.

## 2. Geometry

All dimensions in millimetres. Axisymmetric tool profiles are modelled as analytical rigid
surfaces of revolution; only the blank is deformable.

### Blank
| Parameter | Value |
|---|---|
| Diameter | 112.5 (quarter modelled, R = 56.25) |
| Thickness | 1.0 |
| Material | Aluminium-killed drawing steel |

### Die (analytical rigid, revolute)
| Parameter | Value |
|---|---|
| Cone half-angle from axis | 30° |
| Entry (top) fillet radius | 10.0 |
| Exit fillet radius into land | 5.0 |
| Throat / land radius | 24.5 |
| Land length | 20.0 |

### Punch (analytical rigid, revolute)
| Parameter | Value |
|---|---|
| Punch radius | 22.5 |
| Nose radius | 6.0 |
| Stroke | 123.0 |

### Derived
| Quantity | Value |
|---|---|
| Drawing ratio (blank Ø / punch Ø) | 2.50 |
| Radial clearance (die throat − punch) | 2.00 (2.0 t) |
| Cone angle to sheet plane | 60° |

## 3. Material model

**Elasticity** — isotropic, E = 209 000 MPa, ν = 0.3.

**Hardening** — isotropic, tabulated at 256 points from 131.2 MPa at zero plastic strain to
488.4 MPa at ε_p = 0.51. The table is an exact discretisation of a Swift law:

```
sigma = 572.4 * (0.002 + eps_p)^0.237     [MPa]
```

**Anisotropic yield** — Hill (1948) quadratic potential, defined in Abaqus by the ratios

| R11 | R22 | R33 | R12 | R13 | R23 |
|---|---|---|---|---|---|
| 1.0 | 1.0256 | 1.252 | 1.0897 | 1.0 | 1.0 |

Back-substituting these gives the Lankford coefficients:

```
r0 = 1.91    r45 = 1.48    r90 = 2.23
r_bar = 1.78   (normal anisotropy)
dr    = 0.59   (planar anisotropy)
```

An `r_bar` near 1.8 is characteristic of a well-textured Al-killed drawing sheet and is what
makes a 2.5 drawing ratio feasible at all. The positive `dr` with `r0`, `r90 > r45` predicts
four ears developing at 0° and 90° to the rolling direction.

## 4. Finite element model

| Item | Value |
|---|---|
| Symmetry | Quarter model (two orthogonal symmetry planes) |
| Element type | C3D8R (8-node hex, reduced integration, hourglass control) |
| Elements (blank) | 3 465 |
| Layers through thickness | 3 |
| Nodes (user defined) | 4 868 |
| Total elements incl. contact | 12 707 |
| Total DOF | 42 336 |
| Mesh technique | Swept, advancing front, with a central circular partition for grading |

### Contact
Two surface-to-surface pairs, blank as slave against each analytical rigid master:

- `blank ↔ punch`
- `blank ↔ die` (with initial `adjust = 0.0` to close geometric gaps)

Normal behaviour is hard contact; tangential behaviour is isotropic Coulomb friction with
**μ = 0.15** and a slip tolerance of 0.005. Because the masters are analytical surfaces,
Abaqus falls back from path-based to state-based tracking, which is expected and reported in
the `.dat` file.

### Boundary conditions
| Set | Constraint |
|---|---|
| Blank symmetry face 1 | u1 = 0 |
| Blank symmetry face 2 | u3 = 0 |
| Die reference point | ENCASTRE (fully fixed) |
| Punch reference point | All DOF fixed except u2 = −123.0 |

### Step
Single `*Static` step, `nlgeom=YES`, period 1.0, initial increment 0.001, minimum 1e−5,
maximum 1.0, increment cap 1000. Field and history output at PRESELECT defaults.

## 5. Solution and diagnostics

The job ran to completion on Abaqus/Standard 2021.

| Metric | Value |
|---|---|
| Status | Completed successfully, 0 errors |
| Increments | 304 |
| Automatic cutbacks | 82 |
| Total iterations | 2 170 |
| Severe discontinuity iterations | 1 505 |
| Equilibrium iterations | 665 |
| Wall clock | 1 451 s (≈ 24 min) |
| CPU time | 1 375.5 s |
| Warnings (analysis) | 124, of which 5 negative-eigenvalue |

Roughly 69 % of all iterations were spent resolving contact status changes rather than
material equilibrium. That ratio, together with the 82 cutbacks, is the signature of an
implicit forming analysis where the contact patch sweeps continuously across the die cone
and punch nose. There were also warnings for excessive strain increments and for excessive
distortion at 5 integration points late in the stroke, both worth checking in the deformed
mesh before trusting local thickness values.

## 6. Files

| File | Contents |
|---|---|
| `tdr_dr_pt_new.cae` | Abaqus/CAE model database |
| `tdr_dr_pt_new.jnl` | CAE journal (full Python replay of the model build) |
| `Job-1.inp` | Solver input deck |
| `Job-1.odb` | Output database (~234 MB) |
| `Job-1.dat` | Input processing, model size, preprocessing warnings |
| `Job-1.msg` | Increment-by-increment solver messages |
| `Job-1.sta` | Convergence history summary |
| `Job-1.log`, `.com`, `.prt`, `.sim`, `.ipm` | Execution and restart artefacts |

## 7. Reproducing the run

```bash
# rebuild the model from the journal (headless)
abaqus cae noGUI=tdr_dr_pt_new.jnl

# or solve the deck directly
abaqus job=Job-1 input=Job-1.inp cpus=4 interactive
```

Requires Abaqus 2021 or later with a Standard licence (the original run checked out 5
tokens).

## 8. Post-processing checklist

The quantities this model is set up to answer, extractable from `Job-1.odb`:

1. **Punch load–stroke curve** — reaction force RF2 at the punch reference point against
   punch displacement. Peak load sizes the press.
2. **Thickness strain** — LE33 (or nodal thickness) along a radial path from pole to rim.
   The minimum sits near the punch nose; compare against a thinning limit of about 20 %.
3. **Earing profile** — cup rim height as a function of angle from the rolling direction.
   Only 0° to 90° is available from the quarter model, which is sufficient given the
   orthotropic symmetry.
4. **Equivalent plastic strain PEEQ** — peak value and location, checked against the
   material's forming limit.
5. **Contact pressure CPRESS** on the die cone — indicates whether flange support is
   sufficient to suppress wrinkling.
6. **Wrinkle detection** — visual inspection of the flange in the deformed shape; a
   circumferential compressive LE with no blank holder is the failure mode this geometry is
   designed to avoid.

## 9. Known limitations

- Isotropic hardening only; no kinematic component, so bending–unbending over the die
  radius is not reversed accurately.
- No damage or forming limit criterion, so fracture is not predicted, only inferred from
  thinning.
- Friction is a single constant μ, independent of pressure, speed, and location.
- Three elements through the thickness is adequate for membrane and thinning response but
  coarse for through-thickness bending stress over the 5 mm exit fillet.
- Rate and temperature effects are ignored; the analysis is quasi-static and isothermal.
- Tools are perfectly rigid, so tool deflection and elastic springback of the die are
  excluded.

## 10. Suggested extensions

- Parametric sweep on cone angle (20°/30°/40°) and drawing ratio to map the safe forming
  window between wrinkling and fracture.
- Sensitivity study on μ from 0.05 to 0.20 to quantify the lubrication requirement.
- Comparison against a conventional flat die plus blank holder at equal drawing ratio.
- Springback step after punch retraction to predict final cup diameter.
