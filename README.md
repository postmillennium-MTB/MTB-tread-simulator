# MTB-tread-simulator
interactive MTB tread performance simulator widget
https://www.mdpi.com/2076-3417/10/9/3156
"Characterization and Modelling of Various Sized Mountain Bike Tires and the Effects of Tire Tread Knobs and Inflation Pressure"

Here's the full interactive simulator. Here's what's packed in:

Four live-updating visualizations:

Fig. 6 — Fy vs Slip Angle: The Pacejka Magic Formula Fy = D·sin(C·atan(B·α − E·(B·α − atan(B·α)))) plotted in full, comparing your current configuration (knobbed or smooth) against a slick reference overlay
Fig. 6 — Mz vs Slip Angle: The FastBike model — pneumatic trail t(α) = t₀·cos(Cₜ·atan(Bₜ·α)) multiplied by Fy, plus a polynomial residual c₀ + c₁α + c₂α² — combined exactly as the paper describes
Fig. 5 — Contact Footprint: Canvas-drawn patch with dimensions (length × width in mm), contact area in cm², knob pattern rendered geometrically with staggered rows, and camber-induced lateral shift
Fig. 3 — Pressure Sensitivity Envelope: All five pressure curves (20–40 psi) overlaid simultaneously, with your current pressure highlighted in orange so you can see how the Fy envelope shifts

Live sidebar outputs:

All four Pacejka B·C·D·E coefficients, BCD (cornering stiffness), and αₚₑₐₖ — updating with every slider move
FastBike polynomial coefficients c₀, c₁, pneumatic trail t₀ and Bₜ
Derived outputs: contact area, patch dimensions, peak Fy, peak Mz, μy, and cornering stiffness in N/°

Controls: Wheel size (26/27.5/29"), tire width, inflation pressure (15–50 psi), normal load Fz (200–1000 N), camber angle (±25°), and knob toggle
