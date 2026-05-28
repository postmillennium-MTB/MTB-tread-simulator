

# MTB Tread Performance Simulator

A simplified, interactive physics model for exploring how mountain bike tire
tread compound, knob profile, tire pressure, rider weight, speed, and trail
conditions interact to produce performance outcomes across six scored metrics.

Built as a React artifact using Recharts for visualization.

---

## Purpose

This simulator is a conceptual engineering tool — not a certified tire-testing
instrument. Its goal is to make the *relationships* between tire variables
intuitive and explorable. Changing one input should produce changes in every
output that feel physically plausible, and the charts should reveal trade-offs
that are otherwise invisible when looking at a single spec sheet.

---

## Terrain Variables

Four terrain types are modeled. Each carries three fundamental properties:

| Terrain     | `baseFriction` (μ) | `roughness` | `moistSens` |
|-------------|------------------:|------------:|------------:|
| Rocky       |              0.65 |        0.90 |        0.35 |
| Dirt        |              0.78 |        0.45 |        0.82 |
| Mud         |              0.38 |        0.15 |        1.00 |
| Slick Rock  |              0.52 |        0.08 |        0.98 |

**`baseFriction` (μ)** — The Coulomb friction coefficient for dry tire-to-surface
contact. Dirt is highest because loose-over-hardpack allows knobs to key into
the substrate. Mud is lowest not because the surface is hard but because the
water-saturated matrix offers very little shear resistance before flowing.
Slick rock is intermediate — high static friction on dry sandstone, but
extremely sensitive to moisture film.

**`roughness`** — A macro surface irregularity index from 0 (glass) to 1
(jagged angular boulders). It primarily drives two things: (1) how badly the
tire bounces at speed (rough = more bounce = more speed penalty), and (2) how
quickly the rubber ablates (rough = faster knob wear = lower durability score).

**`moistSens`** — How severely moisture degrades grip on this terrain. Slick
rock approaches 1.0 because even a thin film of water turns its friction from
high to nearly zero; the surface is nonporous and offers no drainage path.
Rocky terrain is low because the micro-texture of angular granite is large
enough to penetrate a thin water film and maintain mechanical interlocking.

---

## Compound Variables

Three rubber compounds are modeled. Each carries five properties:

| Compound | `gripMult` | `durBase` | `rollBase` | `wetAdapt` | `vibDamp` |
|----------|----------:|----------:|-----------:|-----------:|----------:|
| Soft     |      1.35 |      0.38 |       0.72 |       0.22 |      0.72 |
| Medium   |      1.00 |      0.68 |       1.00 |       0.12 |      1.00 |
| Hard     |      0.72 |      1.00 |       1.25 |       0.05 |      1.38 |

**`gripMult`** — Multiplies the terrain's base friction coefficient. Soft
compounds have a lower Shore A durometer, meaning the rubber deforms around
surface micro-texture at the molecular scale, increasing real contact area and
thus grip. Hard compounds deform less, so only macro-scale contacts contribute.

**`durBase`** — Baseline abrasion resistance. Harder compounds resist the
cutting and tearing action of rough terrain better. Softer compounds lose
material faster but also run cooler in cyclic loading scenarios.

**`rollBase`** — Rolling resistance scalar. Soft compounds have high hysteretic
damping — they deform and recover with each knob contact cycle, converting
kinetic energy to heat. Hard compounds spring back more efficiently. A higher
value means *more* rolling resistance (less efficient).

**`wetAdapt`** — Sipe and micro-drainage effectiveness. Soft compounds
physically conform around water-film irregularities and sipes (the small slits
cut into knobs) flex more aggressively to pump water. Hard compounds barely
open their sipes under normal load.

**`vibDamp`** — Vibration damping coefficient. **This is the key fix from v1.**

- Soft (0.72): A soft compound deforms around terrain irregularities at speed,
  maintaining continuous contact. Think of it as a passive suspension layer.
  The speed penalty is *reduced* by 28%.
- Medium (1.00): Baseline. Speed penalty applies as calculated.
- Hard (1.38): A hard compound cannot deform fast enough to follow terrain
  texture at speed. The tire skips and bounces, losing intermittent contact.
  The speed penalty is *amplified* by 38%.

This is why the "Speed Penalty" value in the coefficient table changes when you
switch compounds even at the same speed setting.

---

## Derived Factors (Calculated Each Frame)

These are intermediate values computed from the slider inputs before they
feed into the score equations.

### Pressure Factor (`pF`)
```
pF = 1.30 - ((pressure - 18) / 17) × 0.60
```
At 18 PSI → pF = 1.30 (low pressure, large contact patch)
At 35 PSI → pF = 0.70 (high pressure, small contact patch)

A larger contact patch spreads load across more knobs, increasing total
friction force and compliance. It also increases rolling resistance because
more rubber is deforming per revolution.

### Weight Factor (`wF`)
```
wF = 0.80 + (weight / 250) × 0.40
```
Heavier riders increase the normal force pressing the tire into the terrain.
By Coulomb's law (Friction = μ × Normal Force), this increases absolute grip
up to the point of knob shear failure. The range (0.80–1.12 for 120–250 lbs)
is intentionally modest because the contact patch area also increases,
partially distributing the added load.

### Knob Factor (`kF`)
```
kF = [0.70, 1.00, 1.30] for [Low, Medium, High]
```
Taller knobs penetrate deeper into deformable surfaces before reaching
shear resistance, increasing grip in loose conditions. They also carry
more material in their inter-knob channels for mud displacement. However,
tall knobs on hard surfaces flex more under lateral load, reducing cornering
precision. They also wear faster.

### Speed Penalty (compound-aware)
```
spdBase    = min(0.28, (speed - 5) / 100)
spdPenalty = spdBase × c.vibDamp
```
The raw penalty increases linearly from 5 mph (no penalty) to approximately
30 mph (maximum penalty of 0.28). This is then multiplied by the compound's
vibration damping coefficient. **In v1 this was compound-agnostic — a bug.**
The result is that a soft compound at 25 mph has a meaningfully lower speed
penalty than a hard compound at the same speed.

### Effective Grip Coefficient (`μ_eff`)
```
μ_eff = baseFriction × gripMult × pF
```
This is the single most important derived value. It combines terrain contact
quality, compound deformation behavior, and contact patch size into one number
that feeds directly into traction, cornering, and braking calculations. This
is what the coefficient panel highlights in amber — it changes with terrain
selection *and* compound selection *and* pressure.

---

## Score Equations

All scores are clamped to [5, 100].

### Traction
```
tr = μ_eff × wF
if rain:  tr ×= (1 - wet×0.50 + wetAdapt×0.80)
if mud:   tr ×= (0.45 + kF×0.35)
tr = clamp((tr - spdPenalty) × 100, 5, 100)
```
The mud modifier reflects that on mud, grip comes almost entirely from knob
penetration and displacement rather than surface friction. Tall knobs on mud
disproportionately outperform short knobs.

### Cornering
```
co = baseFriction × 0.88 × gripMult × pF
if rain: co ×= (1 - wet×0.48 + wetAdapt×0.70)
co += (kF - 0.70) × 0.10    ← side knob shoulder contribution
co = clamp((co - spdPenalty×1.5) × 100, 5, 100)
```
Cornering carries a 1.5× speed penalty multiplier because lateral grip
degrades faster than longitudinal grip at speed — the tire wants to track
straight under high centripetal loads.

### Braking
```
br = baseFriction × 0.93 × gripMult × pF × wF
if rain: br ×= (1 - wet×0.55 + wetAdapt×0.90)
br = clamp((br - spdPenalty×2.0) × 95, 5, 100)
```
Braking carries the highest speed penalty multiplier (2.0×) because braking
forces are predominantly longitudinal and high deceleration loads concentrate
force at the front contact patch, which is already stressed. The 0.95 ceiling
reflects that no real-world braking event extracts 100% of theoretical grip.

### Rolling Efficiency
```
re = 1 / (rollBase × (1 + roughness×0.40) × (1 + (1-pF)×0.35))
re = clamp(re × 90, 5, 100)
```
Higher values mean less rolling resistance (more efficient). It's inversely
proportional to the compound's rolling resistance scalar, terrain roughness
(more impacts = more hysteresis cycles), and contact patch size (lower
pressure = more deformation per revolution).

### Durability
```
du = durBase × (1 - roughness×0.38) / max(0.6, kF×0.70)
if rain: du ×= 0.88
du = clamp(du × 110, 5, 100)
```
Hard compounds on smooth terrain (slick rock) score highest. Soft compounds
on rocky terrain score lowest. Taller knobs reduce durability because they
carry more rubber in a thinner cross-section and flex more per loading cycle,
accelerating fatigue. Wet conditions accelerate compound degradation slightly.

### Mud Clearance
```
mc = kF×0.55 + (1 - min(1.35, gripMult)/1.50)×0.25 + max(0, pF-1)×0.10
if rain: mc = min(1.0, mc × 1.15)
mc = clamp(mc × 80, 5, 100)
```
Mud clearance is driven by knob height (tall knobs create wider inter-knob
channels that self-clean under centrifugal force), compound firmness (a
harder compound's wider spacing ejects mud more readily than a soft compound's
close-packed, sticky knobs), and pressure (very low pressure widens the
contact zone and can trap more mud). Rain increases mud clearance demand and
the score reflects how well the current setup handles it.

---

## Overall Score

A single weighted aggregate is computed per terrain type, because the relative
importance of each metric differs by trail:

| Metric      | Rocky | Dirt | Mud  | Slick Rock |
|-------------|------:|-----:|-----:|-----------:|
| Traction    |  0.30 | 0.28 | 0.18 |       0.35 |
| Cornering   |  0.20 | 0.20 | 0.12 |       0.25 |
| Braking     |  0.30 | 0.22 | 0.15 |       0.30 |
| Efficiency  |  0.05 | 0.15 | 0.05 |       0.05 |
| Durability  |  0.10 | 0.10 | 0.10 |       0.04 |
| Mud Clear   |  0.05 | 0.05 | 0.40 |       0.01 |

Mud Clearance dominates the mud trail because the tire that can't self-clean
becomes useless regardless of other properties. Rolling Efficiency matters
more on dirt (XC-style long efforts) than on rocky or slick rock terrain
where survival grip is paramount.

Verdict thresholds: OPTIMAL ≥ 72 · ADEQUATE ≥ 58 · COMPROMISED ≥ 44 · POOR CHOICE < 44

---

## Visual Panels

### Performance Radar
Spider chart of all six metrics for the current setup. Gives a quick shape
signature — a wide, even hexagon is a well-rounded setup; a narrow shape
indicates a specialist tire that sacrifices multiple metrics for one priority.

### Metric Scores
Horizontal bar chart of all six metrics with color coding: green ≥ 70,
amber 45–69, red < 45. The transition animations on the bars make it easy to
see which metrics move most when a single variable changes.

### Compound Comparison (Bar Chart)
Shows Traction, Cornering, Braking, and Efficiency for all three compounds
simultaneously under current trail/conditions/sliders. The active compound is
highlighted at full opacity; inactive compounds are dimmed. Toggling rain or
changing pressure will reshape all three bars, illustrating how conditions
interact differently with each compound.

### Traction & Efficiency vs PSI (Line Chart)
Sweeps pressure from 18 to 35 PSI while holding all other inputs fixed.
A dashed vertical reference line marks the current PSI setting. The crossing
of the Traction and Efficiency lines (if it occurs) indicates the PSI range
where you're trading rolling speed for grip — a classic setup trade-off.

### Grip Degradation vs Speed (Line Chart)
Sweeps speed from 5 to 30 mph with all other inputs fixed. Because `spdPenalty`
is now compound-aware, switching between Soft and Hard will visibly change the
*slope* of these degradation curves, not just their absolute values.

### Contact Patch Visualizer
An SVG proxy showing how contact patch size changes with pressure (width) and
rider weight (height). Knob dots inside the patch reflect compound density —
soft compound packs knobs tighter (more dots), hard compound has wider
inter-knob spacing (fewer, spaced-out dots). This is not to scale but
communicates the directional relationship correctly.

### Active Coefficients Table
Displays all derived factors for the current state. Compound-dependent values
(grip multiplier, vib damping, speed penalty, wet adaptation) are highlighted
in the compound's accent color so it's immediately clear which rows change
with compound selection vs. terrain vs. rider inputs.

---

## Known Simplifications & Limitations

1. **No temperature modeling.** Rubber compounds change dramatically in
   stiffness below 5°C and above 35°C. `gripMult` is treated as a fixed
   constant here.

2. **No lateral vs. longitudinal knob differentiation.** Real tires have
   directional tread patterns where center knobs and side knobs serve
   fundamentally different mechanical roles. This model uses a single `kF`
   multiplier for all knob geometry.

3. **No tire casing model.** Casing stiffness (TPI count, bead type) affects
   damping and sidewall compliance independently of compound. Ignored here.

4. **Linear moisture scaling.** Real wet-grip degradation is nonlinear and
   threshold-dependent — a very light mist has almost no effect, but a
   saturation event can drop grip by 70% suddenly. This model applies a
   smooth scalar.

5. **No rider skill term.** In practice, tire choice interacts with riding
   technique. A hard compound under a skilled rider may outperform a soft
   compound under a beginner.

6. **Contact patch is a proxy.** The SVG contact patch visualizer uses
   pixel units scaled from the pressure and weight factors. Actual contact
   patch dimensions depend on casing compliance, rim width, and tire volume
   in ways not captured here.

---

## Potential Future Extensions

- Temperature slider with compound-specific stiffness curves (DMA data)
- Front vs. rear tire configuration (different knob heights, different tread
  directions) with weight distribution model
- Wear simulation over time (km ridden) degrading durability and grip scores
- Trail segment sequence player — define a 20-section trail and animate the
  contact patch over it, scoring section by section
- Import LiDAR trail roughness profiles as CSV to replace the four static terrain types
- Multi-objective Pareto front visualization: plot all compound×knob×pressure
  combinations as points on a traction vs. efficiency scatter plot

---

## Tech Stack

- React (useState, functional components)
- Recharts (RadarChart, BarChart, LineChart)
- SVG for contact patch visualizer and gauge
- Google Fonts: Barlow Condensed + Share Tech Mono
- No external state management, no localStorage, all values computed inline
