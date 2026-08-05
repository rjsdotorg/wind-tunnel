When `wind-tunnel.html` is opened in a modern browser, it runs a **self-contained, interactive 2D wind-tunnel simulation** entirely inside the browser. It does not contact a server or load external libraries. fileciteturn0file0

## What appears

The page displays:

- A rectangular wind-tunnel test section.
- Flow moving **from right to left**.
- A starter cylinder placed near the center.
- Live drag coefficient \(C_d\) and lift coefficient \(C_l\) meters.
- Controls for speed, visualization mode, simulation resolution, drawing tools, and body shapes.

The default speed slider is 45%, corresponding to approximately:

- **68 mph**
- lattice velocity `U = 0.077`

The “mph” number is a display scale. It is not a physically calibrated conversion to a real full-size tunnel.

## What the simulation does

The code implements a **D2Q9 lattice-Boltzmann method**, or LBM, fluid solver.

“D2Q9” means:

- 2 spatial dimensions.
- 9 discrete particle-flow directions per lattice cell.
- Fluid density and velocity are derived from those nine particle distributions.

The calculation is executed in WebGL2 fragment shaders on the GPU. Each animation frame normally performs several LBM iterations:

```text id="k9e7x2"
Browser animation frame
    ↓
Rasterize bodies into an obstacle bitmap
    ↓
Run approximately 10–20 LBM simulation steps on the GPU
    ↓
Advect the smoke tracer
    ↓
Periodically calculate lift and drag
    ↓
Render the selected visualization
```

At the standard 512 × 256 resolution, it performs 10 fluid iterations per displayed frame. Higher resolutions increase both the lattice dimensions and the number of iterations.

## Startup sequence

When the page loads, the JavaScript:

1. Requests a WebGL2 rendering context.
2. Checks for floating-point render-target support through `EXT_color_buffer_float`.
3. Compiles six GPU shader programs.
4. Allocates the fluid, dye, obstacle, force, and reduction textures.
5. Initializes the tunnel to uniform right-to-left flow.
6. Runs a small GPU benchmark.
7. Selects Low, Standard, High, or Ultra simulation detail automatically.
8. Adds a cylinder as the initial body.
9. Starts the continuous animation loop.

If WebGL2 or floating-point render targets are unavailable, it displays:

> WebGL2 with float render targets is required for this demo and is not available in this browser.

Chrome, Edge, and Firefox on a reasonably modern GPU should generally satisfy those requirements.

## The fluid boundary behavior

The velocity is initialized as:

```javascript id="fq583i"
vec2 u = vec2(-uU, 0.0);
```

The negative X velocity makes the fluid move toward the left.

The boundaries work approximately as follows:

- **Right edge:** fixed incoming freestream.
- **Top and bottom edges:** also held at freestream conditions.
- **Left edge:** copies fluid values from the adjacent interior cell, acting as an approximate outlet.
- **Solid bodies:** use halfway bounce-back, reversing particle populations at solid boundaries.
- **Ground plane:** becomes a stationary solid strip along the bottom.

This is suitable for a visual, comparative demonstration, but it is not equivalent to a high-fidelity CFD boundary-condition treatment.

## Available body shapes

You can add:

- Cylinder
- Box
- Wedge
- Thin plate
- Generic fastback car body
- Airfoils

The airfoil choices are generated mathematically from NACA-style geometry:

- NACA 0016
- NACA 0030
- NACA 2412
- NACA 4412
- NACA 9412
- A custom high-camber “Race wing”

For an airfoil, the leading edge faces the incoming flow from the right.

The **Flip** button reverses the vertical camber. That usually changes positive lift into negative lift or downforce.

## Interacting with bodies

With **Select** active:

- Drag a body to move it.
- Drag the square handle to scale it.
- Drag the round handle to rotate it.
- On touchscreens, two-finger pinch scales it.
- Two-finger twist rotates it.

Keyboard controls after selecting a body:

| Key | Action |
|---|---|
| Arrow keys | Move by 2 lattice cells |
| Shift + arrow | Move by 8 cells |
| `r` | Rotate 5° |
| `Shift+r` / `R` | Rotate −5° |
| `f` | Flip vertically |
| `+` | Increase size 10% |
| `-` | Decrease size about 9% |
| Delete or Backspace | Remove body |

Multiple bodies are treated as one combined obstacle field for the force calculation.

## Freehand drawing

The **Draw** tool paints arbitrary solid obstacles into a hidden canvas.

The **Erase** tool removes only that freehand material. It does not erase the predefined cylinder, box, airfoil, or other object types.

The painted image is converted into the same obstacle texture used by the GPU fluid solver.

## Visualization modes

### Smoke

A passive tracer is injected in horizontal stripes near the right edge. It is transported by the simulated velocity field.

This makes streamlines, wakes, separation, and recirculation visually apparent, although it is technically an advected dye field rather than physical smoke particles.

### Speed

The color indicates local velocity magnitude:

- Dark: slow flow
- Blue/cyan: moderate flow
- Amber/white: high flow

The display normalizes velocity relative to the selected freestream speed.

### Vorticity

This estimates local two-dimensional curl from neighboring velocity cells.

- Orange indicates one rotation direction.
- Blue indicates the opposite direction.
- Dark regions have relatively little rotation.

This view makes separated shear layers and vortices easier to see.

## Lift and drag measurement

The code calculates forces using a **momentum-exchange method** at fluid cells adjacent to solid cells.

Every six display frames it:

1. Computes the X and Y momentum transferred to obstacle boundaries.
2. Stores the force contribution for every lattice cell.
3. Repeatedly reduces the complete texture down to one pixel.
4. Reads that final GPU pixel back into JavaScript.
5. Converts the force into approximate coefficients.

The normalization is:

\[
q = \frac{1}{2} U^2 h
\]

where:

- \(U\) is lattice freestream velocity.
- \(h\) is the combined obstacle’s frontal height in lattice cells.

It then calculates approximately:

\[
C_d = \frac{-F_x}{q}
\]

\[
C_l = \frac{F_y}{q}
\]

The displayed values are smoothed by applying only 10% of each new measurement:

```javascript id="cb2nrk"
state.cd += 0.10 * (newCd - state.cd);
state.cl += 0.10 * (newCl - state.cl);
```

This prevents the meters from jumping excessively.

Force arrows are also drawn at the obstacle field’s approximate centroid:

- Amber horizontal arrow: drag.
- Cyan vertical arrow: lift.

## Important limitation of the coefficients

The displayed \(C_d\) and \(C_l\) are **relative comparison values**, not reliable engineering coefficients for a physical automobile, wing, or wind-tunnel model.

Reasons include:

- The simulation is only two-dimensional.
- Reynolds number is limited by the lattice size and viscosity.
- The speed slider does not create a true real-world Reynolds-number match.
- The tunnel walls and outlet are simplified.
- The body is represented on a relatively coarse pixel lattice.
- Drag is normalized using frontal height rather than a fully defined real-world reference area.
- Ground-plane force is deliberately excluded from the force summation.
- The simulation may still be transient when a reading is observed.

The code itself warns that shapes should be **compared with one another**, rather than treating the values as absolute aerodynamic measurements.

## Speed control

The slider maps 0–100 to:

- 0–150 displayed mph
- lattice velocity 0–0.17

At maximum:

```javascript id="b0qjtb"
U_MAX = 0.17
```

The code comments identify that as a lattice Mach number of approximately 0.29.

Changing speed does not automatically reset the existing flow field. The new inlet speed is applied during subsequent iterations, so the simulation transitions toward the new condition. **Reset flow** immediately reinitializes the complete fluid field at the current speed.

## Detail settings

The available lattice sizes are:

| Setting | Lattice |
|---|---:|
| Low | 384 × 192 |
| Standard | 512 × 256 |
| High | 768 × 384 |
| Ultra | 1024 × 512 |

At startup, Auto benchmarks the GPU and selects the largest resolution projected to keep the simulation portion near a 9 ms frame budget.

Changing resolution:

- Reallocates all GPU simulation textures.
- Scales body positions and dimensions.
- Scales freehand drawing.
- Restarts the flow field.

Ultra detail can consume substantial GPU memory because several full-resolution floating-point textures are maintained simultaneously.

## Buttons

- **Pause:** stops fluid and smoke updates but continues displaying the current result.
- **Run:** resumes simulation.
- **Reset flow:** resets velocity, density, smoke, lift, and drag while preserving bodies.
- **Clear bodies:** removes all predefined bodies and all freehand ink.
- **Ground plane:** adds a stationary solid region along the bottom.
- **Flip:** vertically mirrors the selected body.
- **Delete selected:** removes only the selected predefined body.

## Running it locally

The file should normally run by double-clicking it and opening it in Chrome or Edge:

```text id="6z1zek"
wind-tunnel.html
```

Because it does not fetch external files, opening it through a `file:///...` URL should work. A local web server is not inherently required.

Overall, this is a competent interactive **educational and shape-comparison LBM demonstrator**. It can show wake behavior, separation, qualitative ground effects, and relative force trends, but it should not be treated as validated engineering CFD.
