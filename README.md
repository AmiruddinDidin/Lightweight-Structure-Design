# Preliminary Structural Sizing and Strength Verification of a Semi-Monocoque Wing Box

**Technical documentation for `APSR_Didin Amiruddin_13623037.ipynb`**

Didin Amiruddin (13623037), Kelompok 4

---

## Abstract

This document describes the computational procedure in `APSR_Didin Amiruddin_13623037.ipynb`. The notebook performs the preliminary structural sizing and strength verification of a two-spar, stringer-stiffened wing box at three spanwise stations: the wing root (Hardpoint I), the weapon station (Hardpoint II) and the wing tip. Spanwise distributions of bending moment, shear force and torque at limit load are taken from the load datasheet `DATASHEET_N6` and multiplied by the regulatory factor of safety to obtain ultimate loads. Initial gauges are found from a panel-efficiency formula for the compression cover and from shear-flow theory for the spar webs. The resulting sections are then idealised into discrete booms and checked against material yield, skin buckling, local crippling, stringer column instability and spar-web shear buckling. For each section of the notebook, this document gives the theoretical basis, the governing equations, the assumptions made and the relevant literature.

---

## Table of Contents

1. [Usage Guide](#1-usage-guide)
2. [Conventions and Notation](#2-conventions-and-notation)
3. [Section-by-Section Methodology](#3-section-by-section-methodology)
   - [§0 Imports and Unit Convention](#0-imports-and-unit-convention)
   - [§1 Configuration](#1-configuration)
   - [§2 Datasheet and Load Diagrams](#2-datasheet-and-load-diagrams)
   - [§3 Airfoil and Wing-Box Geometry](#3-airfoil-and-wing-box-geometry)
   - [§4 Design Loads](#4-design-loads)
   - [§5 Initial Sizing](#5-initial-sizing)
   - [§6 Airfoil Structural Layout](#6-airfoil-structural-layout)
   - [§7 Laboratory Visit Reference Data](#7-laboratory-visit-reference-data)
   - [§8 Stress and Stability Verification](#8-stress-and-stability-verification)
   - [§9 Final Design](#9-final-design)
   - [§10 Wing Structural Mass Estimate](#10-wing-structural-mass-estimate)
   - [§11 ATR72-600 Mass Moments of Inertia](#11-atr72-600-mass-moments-of-inertia)
4. [Principal Assumptions and Limitations](#4-principal-assumptions-and-limitations)
5. [References](#5-references)

---

## 1. Usage Guide

### 1.1 Requirements

| Item | Requirement |
|---|---|
| Interpreter | Python ≥ 3.10 (the notebook metadata specifies 3.11) |
| Packages | `numpy`, `pandas`, `matplotlib` |
| Optional | `openpyxl`, needed only if `DATA_FILE` points to the `.xlsx` datasheet |
| Input file | `DATASHEET_N6.csv` (semicolon-separated) or `DATASHEET_N6.xlsx`, in the same folder as the notebook |

### 1.2 Execution

1. Open `APSR_refined.ipynb` in Jupyter or VS Code and select a Python kernel that has the packages above.
2. Run **Restart Kernel → Run All**. The cells depend on one another in order, so run them from top to bottom.
3. Read the results in this order:
   - the design-load table (§4);
   - the initial-sizing table (§5);
   - the *governing margin per check* table printed for each station (§8 and §9);
   - the summary table comparing the auto-sized and final designs (end of §9);
   - the wing-mass breakdown (§10).

### 1.3 Modifying the Analysis

All user inputs are in the **Configuration** cell (§1). No other cell should need editing for an ordinary design iteration.

| Objective | Parameter(s) to edit |
|---|---|
| Change the final structural gauges | `FINAL_DESIGN_MM` (thicknesses in mm; `stringer = (t, e, h)`) |
| Add or move an analysis station | `STATIONS`: add a `Station(name, chord, section)`, where `section` is the `Section` identifier in the datasheet. Add a matching entry to `FINAL_DESIGN_MM`. |
| Change material allowables | `AL2024`, `AL7075`, `AL7475`, or the component-to-material map `MAT` |
| Change spar positions or rib pitch | `FRONT_SPAR`, `REAR_SPAR`, `RIB_PITCH` |
| Change stringer count | `N_BOOMS_PER_SURFACE` |
| Change buckling coefficients | `KC_SKIN`, `C_CRIP`, `K_COL` |
| Change the factor of safety | `FACTOR_OF_SAFETY` |

### 1.4 Interpreting the Margin of Safety

Each check reports a margin of safety,

$$
MoS = \frac{\sigma_{allow}}{\sigma_{applied}} - 1 ,
$$

where $MoS \ge 0$ is marked **SAFE** and $MoS < 0$ is marked **FAIL**. This follows the convention of Bruhn [3] and Niu [5]. Because applied stresses are evaluated at *ultimate* load and compared with *yield* allowables, the margins are conservative (see §4 of this document).

---

## 2. Conventions and Notation

All internal calculations use coherent SI units (m, N, Pa, kg). Millimetres and megapascals appear only in user inputs and printed tables. The conversions go through the constants `MM = 1e-3` and `MPA = 1e6`.

| Symbol | Description | Unit |
|---|---|---|
| $c$ | local chord | m |
| $x, y$ | chordwise and vertical coordinates | m |
| $h_f, h_r, \bar h$ | front, rear and mean spar height | m |
| $w$ | wing-box width (spar-to-spar distance) | m |
| $A$ | enclosed wing-box area, $\bar h w$ | m² |
| $M, V, T$ | bending moment, shear force, torque | N·m, N, N·m |
| $L$ | rib pitch | m |
| $t_s, t_{fw}, t_{rw}$ | skin, front-web and rear-web thickness | m |
| $(t, e, h)$ | stringer thickness, flange width, web height | m |
| $E, \nu$ | Young's modulus, Poisson's ratio | Pa, – |
| $\sigma_y, \tau_y$ | allowable normal and shear stress | Pa |
| $q$ | shear flow | N/m |

---

## 3. Section-by-Section Methodology

### §0 Imports and Unit Convention

This cell imports the numerical (`numpy`), tabular (`pandas`) and plotting (`matplotlib`) libraries and defines the unit-conversion constants. Keeping a single unit system removes the scaling errors that tend to appear in mixed-unit engineering scripts.

---

### §1 Configuration

This cell collects every design variable, so the analysis is fully parameterised. It defines three structures:

- **`Station`**: a spanwise analysis station, given by its name, local chord and the datasheet section at which loads are extracted.
- **`Material`**: an isotropic material with allowable stress $\sigma_y$, density $\rho$, modulus $E$ and Poisson's ratio $\nu$. The allowable shear stress follows the Tresca criterion:

$$
\tau_y = \tfrac{1}{2}\,\sigma_y .
$$

- **`FINAL_DESIGN_MM`**: the gauges chosen by the designer, which replace the values from initial sizing in §9.

The materials are assigned as follows:

| Component | Alloy | $\sigma_y$ (MPa) | $\rho$ (kg/m³) |
|---|---|---|---|
| Upper skin | Al 2024 | 345 | 2780 |
| Lower skin, spars, ribs | Al 7075 | 503 | 2810 |
| Stringers | Al 7475 | 421 | 2800 |

A single modulus, $E = 71.7$ GPa with $\nu = 0.33$, is used for all alloys. For design-level work, allowables should be taken from the Metallic Materials Properties Development and Standardization handbook (MMPDS) [12] for the specific temper, product form and thickness.

> **Note on alloy choice.** Al 2024 is normally used on the *lower* (tension-dominated) cover because of its fatigue and damage-tolerance properties. Al 7075 or 7xxx-series alloys are normally used on the *upper* (compression-dominated) cover [5]. The assignment above follows the course specification and should be reviewed.

---

### §2 Datasheet and Load Diagrams

The function `load_datasheet` reads either the `.csv` or the `.xlsx` datasheet. It removes whitespace from column headers and drops duplicated and unnamed columns. The semi-span $b/2$ is taken as the largest spanwise coordinate in the datasheet.

The shear-force, bending-moment and torque diagrams are plotted against span, with the analysis stations shown as vertical lines. In the datasheet these diagrams come from integrating the net distributed load from the tip inboard. The net load is aerodynamic lift less the inertia relief of the wing structure, fuel and main landing gear, with the concentrated store loads added:

$$
V(y) = \int_y^{b/2} w(\eta)\,d\eta + \sum P_i, \qquad
M(y) = \int_y^{b/2} V(\eta)\,d\eta .
$$

This is the standard procedure in Megson [4] and Niu [5].

---

### §3 Airfoil and Wing-Box Geometry

#### 3.1 Airfoil definition

The section is approximated by a NACA four-digit profile with maximum thickness $t/c = 0.18$, maximum camber $m = 0.06$ at $p = 0.40$. This is a stand-in for the NACA 64-618 profile. The thickness distribution and camber line are those of Abbott and von Doenhoff [9]:

$$
y_t = 5\,\tfrac{t}{c}\left(0.2969\sqrt{x} - 0.1260x - 0.3516x^2 + 0.2843x^3 - 0.1015x^4\right)
$$

$$
y_c =
\begin{cases}
\dfrac{m}{p^2}\left(2px - x^2\right), & x \le p \\[2mm]
\dfrac{m}{(1-p)^2}\left[(1-2p) + 2px - x^2\right], & x > p
\end{cases}
$$

The upper and lower surfaces lie at $y_c \pm y_t\cos\theta$, displaced chordwise by $\mp y_t\sin\theta$, where $\theta = \arctan(dy_c/dx)$. Points are distributed with cosine spacing, $x = \tfrac12(1-\cos\phi)$, to resolve the leading- and trailing-edge curvature. The upper surface folds back slightly near the nose, so `_surface_y` interpolates only from the leading-edge minimum onward. This keeps the interpolation abscissa monotonic.

#### 3.2 Wing-box geometry

The front and rear spars sit at $x/c = 0.25$ and $0.75$. This is within the ranges Roskam [1, Part III] gives for transport and military aircraft: front spar about 15–30 % chord, rear spar about 65–75 % chord. The spar heights are measured *vertically* between the surfaces:

$$
h_f = y_u(x_f) - y_l(x_f), \quad h_r = y_u(x_r) - y_l(x_r), \quad
\bar h = \tfrac12(h_f + h_r), \quad w = x_r - x_f, \quad A = \bar h\,w .
$$

The enclosed area $A$ is approximated by a rectangle of mean height $\bar h$. Its only use is in the Bredt–Batho torsion formula (§5).

---

### §4 Design Loads

The datasheet gives **limit loads**, meaning the largest loads expected in service. The flight load factor $n$ is already included in the inertial terms. Ultimate loads follow from the factor of safety prescribed by CS 25.303 / 14 CFR §25.303 [10]:

$$
(M, V, T)_{ult} = 1.5\,(M, V, T)_{lim} .
$$

Absolute values are used, and the sign of bending is handled in §8: positive moment puts the upper cover in compression. The three stations are:

| Station | Datasheet section | Basis for selection |
|---|---|---|
| Root (Hardpoint I) | 0 | maximum $M$, $V$ and $T$ |
| Hardpoint II (weapon) | 224.5 | largest concentrated store load |
| Tip | 380 | outboard representative station |

> **Data-consistency caveat.** The `Chord` column of the datasheet (1.50 m at the root, 0.70 m at the tip) does not match the design chords used in `STATIONS` (2.57 m and 1.16 m). The table in §4 shows both so that the discrepancy can be resolved before results are reported.

---

### §5 Initial Sizing

Initial sizing gives first-estimate gauges from simple, conservative relations. These gauges are then verified in §8.

#### 5.1 Compression cover: panel-efficiency method

The allowable stress for a stringer-stiffened compression panel is estimated from the panel-efficiency concept introduced by Farrar [8] and reproduced by Niu [5]:

$$
\sigma_{allow} = F\sqrt{\frac{N\,E}{L}}, \qquad N = \frac{M}{\bar h\,w},
$$

where $N$ is the end-load intensity (load per unit cover width) and $F$ is the Farrar efficiency factor. The notebook uses the dimensional form

$$
f_b\,[\text{MPa}] = 138 \times 1.02\,\sqrt{\frac{N\,[\text{MN/m}]}{L\,[\text{m}]}} ,
$$

where the constant $138 \times 1.02$ is the course-supplied value of $F\sqrt{E}$. For $E = 71.7$ GPa it corresponds to $F \approx 0.53$, which is within the range Farrar reports for Z-stiffened panels.

#### 5.2 Skin thickness

The equivalent (smeared) cover thickness needed to carry $N$ at $f_b$ is

$$
t_{eff} = \frac{M}{\bar h\,w\,f_b}.
$$

The skin is assumed to carry 65 % of the equivalent thickness, with the rest provided by the stringers. A minimum gauge of 1 mm is also imposed:

$$
t_s = \max\left(0.65\,t_{eff},\ t_{min}\right).
$$

#### 5.3 Spar webs: shear flow

**Torsion.** The Bredt–Batho relation for a single-cell closed section [4] gives

$$
q_T = \frac{T}{2A}.
$$

**Vertical shear.** The vertical shear is shared between the spars in proportion to the square of their heights, which approximates their relative bending stiffness:

$$
q_{V,i} = \frac{V\,h_i^2}{(h_f^2 + h_r^2)\,\bar h}, \qquad i \in \{f, r\}.
$$

**Web thickness.** The two shear flows are added *on both webs*. In reality the torsional shear flow adds on one web and subtracts on the other, so adding it to both is conservative:

$$
t_{w,i} = \frac{q_{V,i} + q_T}{\tau_y}.
$$

#### 5.4 Spar caps, stringers and ribs

The following proportions are empirical, as specified in the course, and suitable only for first estimates:

$$
t_{cap,w} = 2t_s, \quad t_{cap,h} = t_{fw} + t_{rw}, \quad
(t, e, h)_{str} = (t_s,\ 2t_s,\ 5t_s), \quad
t_{rib} = \frac{V}{2\bar h\,\tau_y}.
$$

The rib-web relation treats the rib as a shear web that transfers the section shear between the covers. A rigorous rib design would use the crushing loads from cover curvature and the redistribution loads at concentrated-load stations [5].

---

### §6 Airfoil Structural Layout

This section draws each station's cross-section: skin, shaded wing box, spars and stringer positions. It uses exactly the same geometry and stringer layout as the analysis in §8, so the figures match the computed model.

---

### §7 Laboratory Visit Reference Data

This section tabulates the gauges measured on a physical wing specimen during the laboratory visit. These gauges serve as an empirical benchmark for judging whether the computed thicknesses are realistic.

| Component | Item | Value (mm) |
|---|---|---|
| Ribs | flange / web thickness | 2.0 / 2.0 |
| Ribs | rivet pitch / rib pitch | 19 / 500 |
| Skin | thickness | 1.4 |
| Spar | web / flange thickness | 2.3 / 3.9 |
| Stringer | thickness / height / pitch | 2.7 / 39.3 / 89.5 |

---

### §8 Stress and Stability Verification

#### 8.1 Structural idealisation

The section is idealised following Megson [4]:

- **Booms.** `N_BOOMS_PER_SURFACE` stringers are placed uniformly between the spars on each cover. Each has area $A_{str} = (2e + h)\,t$.
- **Skin panels.** Each panel between adjacent booms is treated as a straight thin plate of area $t_s \ell$.
- **Spar webs.** Each web is a vertical plate of area $t_w h$.

All three element types carry direct stress. The neutral axis passes through the section **centroid**:

$$
\bar y = \frac{\sum A_k\,y_k}{\sum A_k}.
$$

This matters because the cambered box is not symmetric about the chord line. The second moment of area about the centroidal axis is

$$
I_{xx} = \sum_{booms} A_b (y_b - \bar y)^2
+ \sum_{panels,\,webs} \frac{t\ell}{3}\left(d_1^2 + d_1 d_2 + d_2^2\right),
$$

where $d_{1,2}$ are the distances of the element end points from $\bar y$. This is the exact second moment of a straight inclined thin plate.

#### 8.2 Direct (bending) stress

The engineering bending formula for symmetric bending about the horizontal axis [4] gives

$$
\sigma_z = -\frac{M\,(y - \bar y)}{I_{xx}}.
$$

For positive (upward) bending this places the upper cover in compression. Unsymmetric-bending terms ($I_{xy}$) are neglected. This is acceptable for a near-symmetric box under predominantly vertical loading.

#### 8.3 Spar-web shear stress

The shear flows of §5.3 are reused, so sizing and verification use the same formula. The web shear stress is

$$
\tau_i = \frac{q_i}{t_{w,i}}.
$$

#### 8.4 Material yield

| Element | Criterion | Allowable |
|---|---|---|
| Stringers | $\lvert\sigma_z\rvert \le \sigma_y$ | Al 7475 |
| Skin at each boom | $\lvert\sigma_z\rvert \le \sigma_y$ | Al 2024 (upper), Al 7075 (lower) |
| Spar webs | $\sigma_{VM} = \sqrt{3}\,\tau \le \sigma_y$ | Al 7075 |

Under compatibility of strain, the skin and the stringer at a given boom carry the same direct stress because they share the same modulus.

#### 8.5 Skin compression buckling

Each skin panel between stringers is treated as a flat plate, simply supported on all edges and loaded in uniaxial compression. Its critical stress follows the classical solution of Timoshenko and Gere [11], as tabulated in ESDU 72019 [6] and Gerard and Becker [13]:

$$
\sigma_{cr} = \frac{k_c\,\pi^2 E}{12(1-\nu^2)}\left(\frac{t_s}{b}\right)^2,
\qquad k_c = 4.0,\quad b = \frac{w}{N_{booms}-1}.
$$

This check is applied **only to panels in compression**. Panel curvature (which raises $\sigma_{cr}$) is neglected, which is conservative.

#### 8.6 Stringer crippling

Crippling (local instability) of the thin-walled stringer section is estimated with the course-supplied relation

$$
\sigma_{cc} = \min\left[C\,E\left(\frac{t}{e}\right)^{0.85},\ \sigma_y\right], \qquad C = 0.56 .
$$

The exponent has the same form as the Gerard method [14]. The crippling stress is **capped at the compressive yield stress**, because a formed section cannot cripple above yield [3, 15]. For a more rigorous estimate, use the Needham method [15] or the Gerard method [14] with the section's actual element widths and edge-support conditions.

#### 8.7 Stringer column buckling

Each stringer is treated as a pin-ended column between ribs ($K = 1$, length $L$ = rib pitch). Its radius of gyration is

$$
r = \sqrt{\frac{I_{str}}{A_{str}}}, \qquad I_{str} = \frac{t h^3}{12} + 2\,e\,t\left(\frac{h}{2}\right)^2 .
$$

The Euler critical stress is

$$
\sigma_e = \frac{\pi^2 E}{(KL/r)^2}.
$$

When $\sigma_e > \sigma_y/2$ the column is in the inelastic range, and the Johnson parabola is used instead [3, 11]:

$$
\sigma_{col} = \sigma_y\left(1 - \frac{\sigma_y}{4\sigma_e}\right).
$$

The contribution of the effective width of skin attached to the stringer is neglected, which is conservative.

#### 8.8 Spar-web shear buckling

The web panel between ribs has dimensions $h \times L$. It is treated as simply supported under pure shear (ESDU 71005 [7]; Timoshenko and Gere [11]):

$$
\tau_{cr} = \frac{k_s\,\pi^2 E}{12(1-\nu^2)}\left(\frac{t_w}{b}\right)^2,
\qquad k_s = 5.35 + 4\left(\frac{b}{a}\right)^2,
$$

where $b = \min(h, L)$ and $a = \max(h, L)$. Requiring the web not to buckle at ultimate load is conservative. Transport-aircraft webs are often designed as *tension-field* or *diagonal-tension* beams, which are allowed to buckle [3, 5].

#### 8.9 Reporting

For each station the notebook prints:

- the centroid position and $I_{xx}$;
- the spar-web shear table;
- a *governing margin* table, which keeps the lowest $MoS$ for each check type.

It also draws the section with the stringer yield margin labelled at each boom. Section 8.1 of the notebook applies this procedure to the auto-sized design from §5.

---

### §9 Final Design

The gauges in `FINAL_DESIGN_MM` replace those from initial sizing. The full verification of §8 is then repeated. A combined table shows the governing margin of every check for both designs, plus a column stating whether the final design is safe for every check. Design iteration consists of changing `FINAL_DESIGN_MM` until every margin is non-negative with a reasonable reserve.

---

### §10 Wing Structural Mass Estimate

The mass of one half-wing is built up component by component using semi-empirical area factors. The half-wing planform area is

$$
S_{1/2} = \tfrac{b}{2}\cdot\tfrac{c_r + c_t}{2}.
$$

| Component | Mass relation | Remarks |
|---|---|---|
| Skin | $m = 2.05\,S_{1/2}\,\bar t_s\,\rho$ | wetted area ≈ 2.05 × planform area (both surfaces) |
| Spars | $m = 0.24\,S_{1/2}\,\bar t_w\,\rho$ | two webs, each ≈ 0.12c high |
| Stringers | $m = N_{str}\,\bar A_{str}\,\tfrac{b}{2}\,\rho$ | $N_{str} = 2(N_{booms} - 2)$, so booms at the spar caps are excluded |
| Ribs | $m = \sum_j 0.04\,c_j^2\,t_{rib}\,\rho$ | $n_{rib} = \mathrm{round}\!\left(\tfrac{b/2}{L}\right) + 1$, $t_{rib} = 2$ mm |

Here $\bar t_s$, $\bar t_w$ and $\bar A_{str}$ are averages over the three stations of the final design, so the estimate updates automatically when the design changes. The full-wing mass is twice the half-wing value.

This is a *bottom-up*, geometry-based estimate. It is **not** one of the Class II statistical wing-weight equations in Roskam [1, Part V] (the Cessna, USAF, General Dynamics and Torenbeek methods). Comparing the two is recommended as a plausibility check. The estimate excludes non-optimum mass (joints, fasteners, cut-outs, sealant, local reinforcement), which typically adds a substantial fraction to the ideal structural mass [5, 16].

---

## 4. Principal Assumptions and Limitations

1. **Station-wise analysis.** Each station is analysed as an isolated cross-section. Taper effects on shear flow and load redistribution at concentrated-load stations are neglected.
2. **Symmetric bending.** $I_{xy}$ and chordwise (drag) bending are neglected.
3. **Conservative margins.** Ultimate stresses are compared with yield allowables. A complete substantiation would compare limit stresses with yield and ultimate stresses with ultimate strength, as required by CS 25.305 [10].
4. **No post-buckling capability.** Skin and web buckling are treated as failure. Post-buckled skin (effective-width) behaviour and tension-field webs are not used [3, 5].
5. **Empirical constants.** The panel-efficiency coefficient, the 0.65 skin-load share, the stringer and cap proportions, the crippling constant and the mass-estimation factors are course-supplied empirical values. They should be confirmed against the course reference.
6. **Airfoil approximation.** The NACA 64-618 section is represented by a cambered four-digit profile, which introduces small errors in spar height.
7. **Data consistency.** The design chords and the datasheet chords disagree (see §4 above). This must be resolved because the loads and the geometry must describe the same wing.
8. **Fatigue, damage tolerance and stiffness** (aeroelastic) requirements are not considered.

---

## 5. References

[1] Roskam, J., *Airplane Design*, Parts I–VIII, DARcorporation, Lawrence, KS. In particular *Part III: Layout Design of Cockpit, Fuselage, Wing and Empennage: Cutaways and Inboard Profiles* and *Part V: Component Weight Estimation*.

[2] Roskam, J., *Airplane Flight Dynamics and Automatic Flight Controls*, Part I, DARcorporation, Lawrence, KS.

[3] Bruhn, E. F., *Analysis and Design of Flight Vehicle Structures*, Jacobs Publishing, Indianapolis, 1973.

[4] Megson, T. H. G., *Aircraft Structures for Engineering Students*, 6th ed., Butterworth-Heinemann (Elsevier), Oxford, 2017.

[5] Niu, M. C. Y., *Airframe Structural Design*, Conmilit Press, Hong Kong, 1988.

[6] ESDU 72019, *Buckling of flat isotropic plates under uniaxial and biaxial loading*, ESDU International, London.

[7] ESDU 71005, *Buckling of flat plates in shear*, ESDU International, London.

[8] Farrar, D. J., "The design of compression structures for minimum weight," *Journal of the Royal Aeronautical Society*, Vol. 53, 1949.

[9] Abbott, I. H., and von Doenhoff, A. E., *Theory of Wing Sections*, Dover Publications, New York, 1959.

[10] European Union Aviation Safety Agency, *Certification Specifications for Large Aeroplanes (CS-25)*, CS 25.301–25.305; equivalently 14 CFR Part 25, §§25.301–25.305.

[11] Timoshenko, S. P., and Gere, J. M., *Theory of Elastic Stability*, 2nd ed., McGraw-Hill, New York, 1961.

[12] Battelle Memorial Institute, *Metallic Materials Properties Development and Standardization (MMPDS)*, Federal Aviation Administration (latest edition).

[13] Gerard, G., and Becker, H., *Handbook of Structural Stability, Part I – Buckling of Flat Plates*, NACA TN 3781, 1957.

[14] Gerard, G., *Handbook of Structural Stability, Part IV – Failure of Plates and Composite Elements*, NACA TN 3784, 1957.

[15] Needham, R. A., "The ultimate strength of aluminum-alloy formed structural shapes in compression," *Journal of the Aeronautical Sciences*, Vol. 21, No. 4, 1954.

[16] Torenbeek, E., *Synthesis of Subsonic Airplane Design*, Delft University Press / Kluwer, 1982.

> **Citation note.** Check the ESDU item numbers and the chapter numbers cited above against the current ESDU index and the edition of each text held by your library before formal submission. Also check the radii-of-gyration values used in §11 against the aircraft-class tables in Roskam Part V.
