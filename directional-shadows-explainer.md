# CSS Directional Lighting & Box Elevation (`light-source` & `box-elevation`)

## Status of this Document
**Type:** CSSWG Explainer & Design Space Exploration  
**Purpose:** This document explores how CSS can support directional and positional lighting for automatic shadow generation. It presents a cohesive baseline proposal (§3–§7) followed by alternative design options and trade-offs across each major axis (§8) to kick off a broader CSSWG design conversation.  
**Related Specifications:**
- [CSS Borders and Box Decorations Module Level 4 (`css-borders-4`)](https://drafts.csswg.org/css-borders-4/#drop-shadows)
- [CSS Backgrounds and Borders Module Level 3 (`css-backgrounds-3`)](https://drafts.csswg.org/css-backgrounds-3/#the-background-attachment)
- [CSS Transforms Module Level 2 (`css-transforms-2`)](https://drafts.csswg.org/css-transforms-2/#perspective)
- [Filter Effects Module Level 1 (`filter-effects-1`)](https://drafts.fxtf.org/filter-effects/#LightSourceDefinitions)
- [CSS Values and Units Module Level 4 (`css-values-4`)](https://drafts.csswg.org/css-values-4/#linked-properties)

---

## 1. Problem & Motivation

### 1.1. Authors Think in "Lighting + Height", Not Low-Level Blur Tuples
In nearly every design system (Material Design, Apple HIG, Fluent, Tailwind), shadows are used to communicate **elevation** ($z$-height of an element above its surface) under a consistent **lighting environment**.

However, `box-shadow` requires authors to manually compute the optical projection of that scene for every component:
```css
/* Today: Authors must hand-calculate offset-x, offset-y, blur, spread, and alpha
   for every elevation step, and combine key + ambient shadow layers manually */
.card-sm {
  box-shadow:
    0px 2px 4px -1px rgb(0 0 0 / 16%),
    0px 1px 3px  0px rgb(0 0 0 / 8%);
}
.card-lg {
  box-shadow:
    0px 12px 24px -6px rgb(0 0 0 / 16%),
    0px  4px 12px -2px rgb(0 0 0 / 8%);
}
```
This has four major pain points for web developers:
1. **Hard to author & maintain:** Changing the angle, softness, or distance of a page's light source requires recalculating every shadow token across the stylesheet.
2. **Collision with `box-shadow` outlines/rings:** Developers frequently use `box-shadow: 0 0 0 2px <color>` for focus rings, hairline borders, or stacked outlines. Because both live in `box-shadow`, adding a focus ring clobbers the element's elevation shadow unless carefully composed via custom properties.
3. **No spatial awareness for local lights:** Making shadows diverge naturally from a finite-distance light source across a grid of cards—or react dynamically as items scroll past a light source—currently requires JavaScript `scroll` and `ResizeObserver` listeners updating inline styles on every frame.
4. **No automatic recessed (`inset`) symmetry:** Creating a sunken well (e.g., an input field or recessed panel) requires manually flipping to `inset` and re-tuning offsets and blurs to match the page's light direction.

### 1.2. Goals
- Allow a container (such as `:root` or a component container) to define **one or more light sources** (`light-source`) with direction, optional finite distance, origin, softness, and color.
- Allow any descendant element to declare its **height above its immediate surface** (`box-elevation`), automatically generating realistic shadows without clobbering explicit `box-shadow` declarations.
- Ensure shadows are **static during scrolling by default**, while providing an explicit opt-in (`light-attachment: scrollport | viewport`) for shadows that dynamically shift as content scrolls past a positional light.
- Reuse the existing `box-shadow` rendering pipeline so `border-radius`, `corner-shape`, `border-shape`, `box-decoration-break`, and knockout clipping work out of the box.

---

## 2. Design Philosophy & Precedents in CSS

This proposal is built directly on established patterns across CSS specifications:

| CSS Precedent | How It Informs Directional Shadows |
| :--- | :--- |
| **`perspective` & `perspective-origin`** (`css-transforms-2`) | Container establishes the 3D scene origin (`light-origin`) and distance; descendants simply declare their $z$-participation (`box-elevation`). Unlike `perspective`, lighting does **not** require `transform-style: preserve-3d` and is not flattened by `overflow` or `opacity`. |
| **`feDistantLight` vs. `fePointLight`** (`filter-effects-1`) | Unifies infinite-distance directional lighting (defined by polar angles) and finite-distance 3D positional lighting into a single continuous model where `infinite` distance is simply the $d \to \infty$ limit. |
| **`background-attachment`** (`css-backgrounds-3`) | Separates content-attached reference frames (static on scroll) from scrollport-attached and viewport-attached reference frames (dynamic on scroll). |
| **Coordinating List Properties** (`css-values-4`, `css-borders-4`) | Just as `box-shadow` and `background` decompose into comma-separated coordinating list longhands, `light-source` decomposes into `light-direction`, `light-softness`, `light-color`, `light-origin`, and `light-attachment`. |

---

## 3. Proposed Properties & Syntax (Baseline Proposal)

### 3.1. Overview of Properties

- **On a Lighting Container (e.g. `:root`, `.panel`):**
  - `light-source` (shorthand)
    - `light-direction`: `<azimuth> <tilt>? <distance>?`
    - `light-softness`: `<percentage> | <number> | <angle> | <length>`
    - `light-color`: `<color>`
    - `light-origin`: `<position>`
    - `light-attachment`: `local | scrollport | viewport`
- **On an Elevated Descendant (e.g. `.card`, `.button`):**
  - `box-elevation`: `<length>` (positive = raised / `outset`; negative = recessed / `inset`)
  - Composes with explicit `box-shadow` (elevation shadow layers are painted immediately behind explicit `box-shadow` layers).

### 3.2. `light-direction`: Azimuth, Tilt, and Optional Distance

```
Name: light-direction
Value: [ none | <azimuth> <tilt>? <distance>? ]#
Initial: none
Applies to: all elements
Inherited: no (establishes a lighting context for descendants; see §4)
Percentages: N/A
Computed value: list, each item either none or a tuple of computed (azimuth, tilt, distance)
Animation type: by computed value
```

Where:
```ebnf
<azimuth>  = <angle> | [ left | right ] || [ top | bottom ]
<tilt>     = <angle [0deg, 90deg)>
<distance> = infinite | <length [0px, ∞]>
```

- **`<azimuth>`**: The compass direction on the $XY$ plane **from which** the light shines, following standard CSS `<angle>` conventions:
  - `0deg` (`top`): light shines from the top edge (casting shadows downward, $+Y$).
  - `90deg` (`right`): light shines from the right edge (casting shadows leftward, $-X$).
  - `180deg` (`bottom`): light shines from the bottom edge (casting shadows upward, $-Y$).
  - `270deg` (`left`): light shines from the left edge (casting shadows rightward, $+X$).
  - Side keywords map to their standard angles (`top` = `0deg`, `top right` = `45deg`, `right` = `90deg`, `top left` = `315deg`, etc.).
- **`<tilt>`**: The zenith angle in `[0deg, 90deg)` measuring how far the light is tilted away from straight overhead ($+Z$ axis). Defaults to `0deg` if omitted.
  - `0deg`: Light is directly overhead (casts a zero-offset ambient shadow when `<distance>` is `infinite`).
  - `45deg`: Light is at a 45° angle, so for an infinite-distance light, shadow offset magnitude equals `box-elevation`.
- **`<distance>`**: The 3D distance $d$ from the container's `light-origin` to the light source along the ray defined by `(<azimuth>, <tilt>)`.
  - Defaults to **`infinite`** if omitted.
  - When `infinite`, light rays are parallel across the entire container (purely directional light).
  - When a finite `<length>` is specified (e.g., `600px`), the light is positioned in 3D space at distance $d$ from `light-origin` (positional light).

### 3.3. `light-origin`: Reference Point on the Container

```
Name: light-origin
Value: <position>#
Initial: 50% 50%
Applies to: all elements
Inherited: no
Percentages: refer to the size of the lighting reference box (border-box)
Computed value: list, each item a pair of computed <length-percentage> offsets
Animation type: by computed value
```

- Sets the $(X_0, Y_0)$ anchor point on the lighting container's border box (at $Z = 0$) from which finite-distance lights are measured—analogous to `perspective-origin`.
- When a light's `<distance>` is `infinite`, `light-origin` has no visual effect on that light because parallel rays have identical angles everywhere.

### 3.4. `light-softness`: Controlling Shadow Blur & Spread

```
Name: light-softness
Value: [ <percentage [0,∞]> | <number [0,∞]> | <angle [0deg, 90deg)> | <length [0,∞]> ]#
Initial: 50%
Applies to: all elements
Inherited: no
Percentages: converted to <number> (100% = 1)
Computed value: list, each item a <number> (for percentage/number/angle) or a computed <length>
Animation type: by computed value
```

`light-softness` controls how rapidly the shadow's penumbra (`blur`) expands as an element's `box-elevation` ($h$) increases:
- **`<percentage>` or `<number>` (Recommended for Web Developers)**: Directly specifies the dimensionless **blur-to-height ratio** $\kappa \ge 0$ (`100%` = `1.0`):
  - `0%` (`0`): Point/collimated light $\implies$ **hard, crisp shadow** (`blur = 0px` at every elevation).
  - `50%` (`0.5`): Moderate softness $\implies$ `0.5px` of blur per `1px` of elevation (at `box-elevation: 10px`, `blur` is `5px`).
  - `100%` (`1`): Soft light $\implies$ `1px` of blur per `1px` of elevation (at `box-elevation: 10px`, `blur` is `10px`).
  - `200%` (`2`): Very diffuse ambient light $\implies$ `2px` of blur per `1px` of elevation.
- **`<angle [0deg, 90deg)>`**: Specifies the angular radius $\delta$ of the light source; computes to the dimensionless ratio $\kappa = \tan(\delta)$.
- **`<length [0,∞]>`**: Specifies a physical light radius $R_L$ (e.g., `100px`).
  - For a finite-distance light at height $Z_L$, the effective angular softness is $\kappa = R_L / Z_L$.
  - If `<distance>` is `infinite`, a `<length>` value for `light-softness` is treated as `0` (since a finite radius at infinite distance subtends an angle of `0deg`).

### 3.5. `light-color`: Shadow Color & Intensity

```
Name: light-color
Value: <color>#
Initial: rgb(0 0 0 / 20%)
Applies to: all elements
Inherited: no
Percentages: N/A
Computed value: list, each item a computed color
Animation type: by computed value
```

Defines the base shadow color cast by each corresponding light source. Defaults to semi-transparent black (`rgb(0 0 0 / 20%)`) so authors get natural shadows without having to repeat colors on every declaration.

### 3.6. `light-attachment`: Scroll Behavior Control

```
Name: light-attachment
Value: [ local | scrollport | viewport ]#
Initial: local
Applies to: all elements
Inherited: no
Percentages: N/A
Computed value: list, each item the specified keyword
Animation type: discrete
```

Determines how finite-distance lights interact with scrolling (see §5 for full details):
- **`local` (Initial)**: The light is affixed to the lighting container's **scrollable content coordinate space**. Scrolling the container moves the content and the light together, so **shadows never change during scrolling**.
- **`scrollport`**: The light is affixed to the lighting container's **border box (scrollport)**. As content scrolls inside the container, elements move underneath the light and **their shadows dynamically shift in real time**.
- **`viewport`**: The light's `light-origin` is resolved against the **viewport** (initial containing block) and stays pinned to the screen. Scrolling *any* scroll container moves elements relative to the viewport light, dynamically updating their shadows.

### 3.7. `light-source`: Coordinating List Shorthand

```
Name: light-source
Value: none | <light-layer>#
Initial: none
Applies to: all elements
Inherited: no
Percentages: see individual properties
Computed value: see individual properties
Animation type: see individual properties
```

Where:
```ebnf
<light-layer> = <'light-direction'>
                && [ / <'light-softness'> ]?
                && <'light-color'>?
                && [ at <'light-origin'> ]?
                && <'light-attachment'>?
```
*(Note: Using `/` before `<'light-softness'>` avoids any ambiguity when `<distance>` and `<'light-softness'>` are both `<length>` values, or when `<azimuth>`, `<tilt>`, and `<'light-softness'>` are all `<angle>` values.)*

`light-source` and its longhands form a coordinating list property group with `light-direction` as the coordinating list base property.

### 3.8. `box-elevation`: Element Height Above Its Surface

```
Name: box-elevation
Value: <length>
Initial: 0px
Applies to: all elements
Inherited: no
Percentages: N/A
Computed value: a computed <length>
Animation type: by computed value
```

- **`box-elevation: 0px` (Initial)**: The element rests flat on its parent surface. No elevation shadow is generated.
- **Positive `<length>` (e.g. `box-elevation: 8px`)**: The element is raised `8px` above its parent surface. Generates one **outset** shadow layer per active light in the nearest ancestor lighting context.
- **Negative `<length>` (e.g. `box-elevation: -4px`)**: The element is recessed/sunken `4px` *into* its parent surface. Generates one **inset** shadow layer per active light in the nearest ancestor lighting context.

> **Composability with `box-shadow`:** Setting `box-elevation` turns on light-driven shadows independently of `box-shadow`. If an element specifies both `box-elevation: 8px` and `box-shadow: 0 0 0 2px royalblue`, both render: the explicit `box-shadow` layers are painted in front, and the `box-elevation` shadow layers are painted immediately behind them.

---

## 4. Optical & Mathematical Model

### 4.1. Establishing the Lighting Context & Accumulating Nested Elevation
1. **Lighting Container Lookup**: An element $E$ with `box-elevation` $h \neq 0$ finds its **lighting container** $C$, which is the nearest ancestor with a computed `light-direction` other than `none`. If no such ancestor exists, no elevation shadow is generated.
2. **Local Drop Height vs. Cumulative 3D Height**:
   - Visually, $E$'s shadow is painted immediately behind $E$'s background—meaning the shadow lands on $E$'s **parent surface**, not on $C$ (unless $C$ is $E$'s parent). Thus, the distance the shadow travels from $E$ to its receiving surface is always $E$'s own local drop height $|h|$.
   - However, in 3D space relative to $C$ ($Z_C = 0$), any intermediate ancestors between $C$ and $E$ that also have `box-elevation` raise the surface itself closer to finite-distance lights.
   - Let the receiving surface height above $C$ be:
     $$Z_{\text{surface}} = \sum_{A \in \text{ancestors}(E, C)} \text{box-elevation}(A)$$
   - Then the shadow-casting plane sits at height:
     $$Z_{\text{caster}} = \begin{cases} Z_{\text{surface}} + h & \text{if } h > 0 \text{ (raised top surface casts shadow onto } Z_{\text{surface}}\text{)} \\ Z_{\text{surface}} & \text{if } h < 0 \text{ (rim at } Z_{\text{surface}} \text{ casts inset shadow onto sunken floor } Z_{\text{surface}} + h\text{)} \end{cases}$$

### 4.2. Converting Each Light into a `box-shadow` Layer

For each light layer $i$ defined on lighting container $C$ with azimuth $\alpha$, tilt $\phi \in [0^\circ, 90^\circ)$, distance $d \in (0, \infty]$, softness $\kappa$, and color $c$:

#### Case 1: Infinite Distance (`d = infinite` — Directional Light)
Light rays are parallel across the entire scene with unit direction vector toward the surface:
$$\vec{u} = \bigl(-\sin(\phi)\sin(\alpha),\; \sin(\phi)\cos(\alpha),\; -\cos(\phi)\bigr)$$

For an element with local elevation $h \neq 0$:
1. **Position (`box-shadow-position`)**:
   $$\text{position} = \begin{cases} \text{outset} & \text{if } h > 0 \\ \text{inset} & \text{if } h < 0 \end{cases}$$
2. **Horizontal and Vertical Offsets (`box-shadow-offset`)**:
   $$\text{offset}_x = -|h| \cdot \tan(\phi) \cdot \sin(\alpha)$$
   $$\text{offset}_y = +|h| \cdot \tan(\phi) \cdot \cos(\alpha)$$
   *(Note: In CSS `box-shadow`, a positive `(offset-x, offset-y)` on an `inset` shadow shifts the cutout down-and-right, casting the inner shadow along the top-and-left edges—which is optically identical to a light from the top-left illuminating a recessed well!)*
3. **Blur Radius (`box-shadow-blur`)**:
   $$\text{blur} = |h| \cdot \kappa$$
4. **Spread Distance (`box-shadow-spread`)**:
   When a light has non-zero softness ($\kappa > 0$), its penumbra bleeds outward on all sides of the box. Contracting the umbra slightly prevents soft shadows from puffing out backwards toward the light source:
   $$\text{spread} = -|h| \cdot \kappa \cdot 0.25$$

#### Case 2: Finite Distance (`d = <length>` — 3D Positional Light)
Let $(X_0, Y_0)$ be the resolved `light-origin` on $C$. The light's 3D coordinates relative to $C$ are:
$$X_L = X_0 + d \cdot \sin(\phi)\sin(\alpha), \qquad Y_L = Y_0 - d \cdot \sin(\phi)\cos(\alpha), \qquad Z_L = d \cdot \cos(\phi)$$

Let $(X_E, Y_E)$ be the center of element $E$'s border box mapped into $C$'s coordinate space (according to `light-attachment`; see §5), and let $\Delta Z = \max(1\text{px},\; Z_L - Z_{\text{caster}})$ be the vertical distance from the light to the shadow-casting edge.

By similar triangles:
1. **Projection Factor**:
   $$k = \frac{|h|}{\Delta Z} = \frac{|h|}{Z_L - Z_{\text{caster}}}$$
2. **Horizontal and Vertical Offsets**:
   $$\text{offset}_x = (X_E - X_L) \cdot k, \qquad \text{offset}_y = (Y_E - Y_L) \cdot k$$
3. **Blur Radius**:
   - If `light-softness` is a dimensionless ratio $\kappa$ (from `<percentage>`, `<number>`, or `<angle>`), the physical light radius is $R_L = \kappa \cdot Z_L$, yielding:
     $$\text{blur} = R_L \cdot k = \kappa \cdot Z_L \cdot \frac{|h|}{Z_L - Z_{\text{caster}}}$$
   - If `light-softness` is a `<length>` $R_L$, then directly:
     $$\text{blur} = R_L \cdot k = R_L \cdot \frac{|h|}{Z_L - Z_{\text{caster}}}$$
4. **Continuous Limit as $d \to \infty$**:
   Observe that as $d \to \infty$ (so $Z_L = d\cos\phi \to \infty$), $\frac{X_E - X_L}{Z_L - Z_{\text{caster}}} \to -\tan(\phi)\sin(\alpha)$, $\frac{Y_E - Y_L}{Z_L - Z_{\text{caster}}} \to +\tan(\phi)\cos(\alpha)$, and $\frac{Z_L}{Z_L - Z_{\text{caster}}} \to 1$.
   **The finite-distance formulas converge to the exact directional light formulas.**

---

## 5. Scrolling Behavior: Static by Default vs. Dynamic on Scroll

A critical requirement for web UI is that **shadows must not unexpectedly wobble or recalculate during scrolling**, while still enabling authors to build scroll-reactive lighting scenes when desired.

### How `(X_E, Y_E)` and `(X_0, Y_0)` Resolve Under Each `light-attachment` Mode

| `light-attachment` | How `light-origin` $(X_0, Y_0)$ is Resolved | How Element Center $(X_E, Y_E)$ is Resolved | Changes on Scroll? | Compositor / Perf Characteristics |
| :--- | :--- | :--- | :--- | :--- |
| **`local` *(Initial)*** | Relative to lighting container $C$'s border-box size, anchored at $C$'s content origin | Layout offset of $E$ relative to $C$, **excluding** any scroll offsets of $C$ or intermediate scrollers | **Never** | Computed once at layout time; zero work during compositor scrolling. |
| **`scrollport`** | Relative to lighting container $C$'s **border box (scrollport)** | Visual offset of $E$ relative to $C$'s border box, **including** scroll offsets of $C$ and intermediate scrollers (matching `css-transforms-2` accumulated 3D matrix) | **Yes**, when scrolling inside $C$ | Can be updated on the compositor thread alongside scroll offsets (just like `perspective-origin`). |
| **`viewport`** | Relative to the **viewport** (initial containing block) | Visual offset of $E$ relative to the viewport, **including** all ancestor and document scroll offsets | **Yes**, when scrolling any ancestor or window | Pinned to screen coordinates (like `background-attachment: fixed`). |

---

## 6. Examples

### Example 1: Design System Dual-Light Setup (Key + Ambient, Static on Scroll)
Define a global key light + overhead ambient light once on `:root`. Components only declare `box-elevation`, leaving `box-shadow` completely free for focus rings and borders:

```css
:root {
  /* Light 1 (Key): shines from top-left (330deg), tilted 25deg, 60% softness
   * Light 2 (Ambient): straight overhead (0deg 0deg), 120% softness, fainter alpha */
  light-source:
    330deg 25deg / 60%  rgb(0 0 0 / 16%),
    0deg   0deg  / 120% rgb(0 0 0 / 8%);
}

.card {
  box-elevation: 6px;
  border-radius: 12px;
  transition: box-elevation 180ms ease;
}

.card:hover {
  box-elevation: 16px; /* Smoothly animates both shadow layers! */
}

/* Focus ring uses box-shadow without clobbering the card's elevation shadow! */
.card:focus-visible {
  box-shadow: 0 0 0 3px oklch(65% 0.2 250);
}

/* Sunken input well inside the card */
.card input {
  box-elevation: -3px; /* Automatically generates a 2-layer inset shadow! */
}
```

### Example 2: Positional Light Across a Bento Grid (Static on Scroll)
A finite-distance light positioned above the top-center of `.bento-grid`. Cards on the left cast shadows to the left; cards on the right cast shadows to the right; nested buttons inside cards sit closer to the light (`8px + 4px = 12px`), and scrolling the page does **not** alter the shadows (`local` attachment by default):

```css
.bento-grid {
  /* Light at top-center (50% 0%), straight above that origin (0deg 0deg) at height 600px,
   * with a physical light radius of 120px */
  light-source: 0deg 0deg 600px / 120px rgb(0 0 0 / 20%) at 50% 0%;
}

.bento-card {
  box-elevation: 8px;
}

.bento-card .action-button {
  /* Casts a 4px drop shadow onto .bento-card, evaluated at Z = 8px + 4px = 12px */
  box-elevation: 4px;
}
```

### Example 3: Scroll-Reactive Showcase (`scrollport` Attachment)
Opting into `scrollport` pins the positional light to the scroll container's frame. As cards scroll from the bottom of the container past the top, their shadows dynamically swing from pointing downward to pointing upward:

```css
.scrolling-gallery {
  overflow-y: auto;
  height: 500px;
  light-source:
    0deg 0deg 450px / 90px rgb(0 0 0 / 22%) at 50% 20% scrollport,
    0deg 0deg / 100% rgb(0 0 0 / 8%); /* Static ambient contact shadow */
}

.gallery-item {
  box-elevation: 10px;
}
```

---

## 7. Interaction with Existing CSS Features

1. **Stacking Order & Painting**:
   - Per CSS Borders 4 § Layering, Layout, and Other Details, shadows are painted front-to-back.
   - Any explicit `box-shadow` layers on an element are painted first (in front), followed by the shadow layers generated by `box-elevation` (in the order the lights are listed in `light-source`).
   - Setting `box-elevation` does **not** establish a stacking context or containing block.
2. **Shape, Corners, and Fragmentation**:
   - Shadow layers generated by `box-elevation` follow the exact same shape rules as `box-shadow`: they conform to `border-radius`, `corner-shape`, `border-shape`, and `box-decoration-break`.
3. **Animations & Transitions**:
   - `box-elevation` interpolates as a `<length>`.
   - `light-direction`, `light-softness`, `light-color`, and `light-origin` interpolate per item in the coordinated list, allowing smooth transitions when moving a light source or toggling dark/light mode shadow intensities.

---

## 8. Design Space: Alternative Options & Trade-Offs

The baseline proposal in §3–§7 represents one coherent point in the design space. However, there are several alternative ways to slice these concepts in CSS. This section lays out the options and trade-offs across six major axes to seed discussion.

### 8.1. Axis 1: Specifying Light Direction vs. 3D Position

How should authors express where the light comes from, and how should infinite-distance (directional) and finite-distance (positional) lights relate?

| Option | Syntax Sketch | Pros | Cons |
| :--- | :--- | :--- | :--- |
| **Option 1A: Unified Polar + Optional Distance *(Baseline)*** | `light-direction: 330deg 25deg 600px?;`<br/>`light-origin: 50% 0%;` | • Smoothly unifies infinite and finite lights (omitting `<distance>` = `infinite`).<br/>• Polar angles are intuitive for directional UI lighting ("top-left, tilted 25°"). | • Placing a finite point light at a specific $(X, Y, Z)$ coordinate typically requires either setting `tilt: 0deg` + `light-origin: X Y` or doing mental trigonometry. |
| **Option 1B: Explicit `directional()` vs. `point()` Functions** | `light-position: directional(330deg 25deg);`<br/>`light-position: point(50% 0% 600px);` | • Self-documenting functional syntax (mirrors SVG `<feDistantLight>` vs. `<fePointLight>`).<br/>• `point(X Y Z)` folds `light-origin` directly into the 3D position (like `transform-origin`). | • Interpolating between `directional()` and `point()` requires special-case rules.<br/>• Slightly more verbose for the common directional case. |
| **Option 1C: Cartesian 3D Position Only (`<position> <length>`)** | `light-position: 50% -1000px 2000px;` | • Only one mental model: a light bulb in 3D space $(X, Y, Z)$ relative to the container.<br/>• Familiar from `transform-origin`. | • Cannot express a true scroll-invariant, container-size-invariant directional light without an `infinite` keyword or huge magic numbers.<br/>• Hard to reason about angles. |
| **Option 1D: Direct 2D Slope / Offset Multiplier** | `light-slope: 0.25 0.5;` *(1px elevation $\to$ 0.25px X, 0.5px Y)* | • Zero trigonometry; web developers immediately know the exact pixel offset produced per pixel of elevation. | • Cannot naturally represent finite-distance lights where shadows diverge from a point on the container. |

### 8.2. Axis 2: How Element Elevation Connects to `box-shadow`

How should an element opt into casting a light-driven shadow, and how should it compose with manual `box-shadow` declarations?

```css
/* Option 2A (Baseline): Independent `box-elevation` property.
 * Automatically appends elevation shadow(s) behind explicit `box-shadow`s. */
.button {
  box-elevation: 4px;
  box-shadow: 0 0 0 2px royalblue; /* Focus ring stays intact! */
}

/* Option 2B: `elevation()` function inside `box-shadow` (or in addition to `box-elevation`).
 * Gives explicit control over front-to-back layer ordering! */
.button {
  box-shadow:
    0 0 0 2px royalblue,
    elevation(4px),
    0 20px 40px rgb(0 0 0 / 5%);
}

/* Option 2C: `box-shadow: auto` keyword driven by `box-elevation`.
 * Explicit `box-shadow` overrides elevation shadow unless `auto` is included. */
.button {
  box-elevation: 4px;
  /* Must include `auto` to keep the elevation shadow when setting box-shadow: */
  box-shadow: 0 0 0 2px royalblue, auto;
}

/* Option 2D: Reuse 3D Transforms (`translateZ` / `translate: 0 0 4px`) instead of `box-elevation`. */
.button {
  translate: 0 0 4px;
  box-shadow: auto;
}
```

- **Trade-offs:**
  - **Option 2A (`box-elevation` auto-appends behind `box-shadow`)** is the most ergonomic for design systems because utility classes for rings/outlines (`box-shadow`) and elevation (`box-elevation`) never clobber each other. However, there is no way for `box-shadow: none` alone to suppress an ancestor/utility `box-elevation` (one must set `box-elevation: 0px`), nor can an explicit `box-shadow` be layered *behind* the elevation shadow.
  - **Option 2B (`elevation(<length>)` function in `box-shadow`)** can actually be **combined with Option 2A**: `box-elevation: 4px` works standalone, while an `elevation()` function in `box-shadow` (or `box-shadow-offset: elevation(4px)`) lets advanced authors interleave manual and light-driven shadow layers in any order.
  - **Option 2C (`box-shadow: auto`)** matches how CSS often links properties, but brings back the exact developer complaint that setting `box-shadow` for an outline accidentally wipes out the elevation shadow.
  - **Option 2D (Reuse `translateZ` / `translate`)** avoids introducing `box-elevation`, but in `css-transforms-2` any non-`none` transform creates a stacking context and a containing block for `position: fixed` descendants, and visibly scales the element if an ancestor has `perspective`. Developers almost never want a card shadow to break `position: fixed` modals or scale the card's layout size.

### 8.3. Axis 3: Single Light with Built-In Ambient Shadow vs. Multi-Light Lists

In real UI design, a single directional shadow often looks harsh on the light-facing edge unless paired with a subtle overhead ambient/contact shadow.

- **Option 3A: Comma-Separated List of Lights *(Baseline)***
  - `light-source` accepts `#`-separated lights (just like `box-shadow` and `background`). Authors explicitly declare a directional key light and an overhead (`0deg 0deg`) ambient light.
  - *Pros:* Completely predictable mapping (1 light = 1 shadow layer); supports arbitrary artistic setups (e.g., warm key light + cool rim light).
  - *Cons:* Requires specifying two lights in `:root` for the standard key + ambient look.
- **Option 3B: Single Light + Built-In `light-ambient` Property**
  - Inspired by Android's `RenderNode` lighting model (which pairs a single virtual 3D light with a global `ambientShadowAlpha` and `spotShadowAlpha`):
  ```css
  :root {
    light-source: 330deg 25deg / 60% rgb(0 0 0 / 16%);
    light-ambient: 120% rgb(0 0 0 / 8%); /* Automatically adds an overhead contact shadow */
  }
  ```
  - *Pros:* Expresses the domain concepts ("directional key light" vs. "omnidirectional ambient occlusion") directly.
  - *Cons:* Easily simulated by a second `0deg 0deg` entry in Option 3A's comma-separated list without needing another property.

### 8.4. Axis 4: Container Scoping & Inheritance Model

How do descendants find their active `light-source` and the reference box used to resolve `light-origin` / finite distances?

- **Option 4A: Implicit Lighting Reference Box on Declaration *(Baseline)***
  - Any element with `light-source != none` establishes the lighting reference box for its subtree.
  - *Pros:* Zero boilerplate; setting `light-source` on `:root` or `.bento-grid` just works.
  - *Cons:* In CSS engines, inherited properties normally pass down computed values without tracking *which* ancestor element declared them (unless backed by a container/context lookup similar to `container-type` or `anchor-scope`).
- **Option 4B: Standard Inherited `light-*` Properties + Explicit `light-scope` (or `container-type`)**
  - `light-source` is a standard inherited property. For infinite-distance lights, no reference box is needed! For finite-distance lights, `light-origin` resolves against the nearest ancestor with `light-scope: box` (or defaults to the viewport / initial containing block if none is set).
  - *Pros:* Pure CSS value inheritance for the 90% case (directional lights); explicit container tracking only when finite-distance container-local lights are used.
- **Option 4C: CSS Anchor Positioning Integration (`anchor()`)**
  - Allow `light-origin` to reference a named anchor element (e.g., `light-origin: anchor(--lamp center)`), so a visible "lamp" or highlight element on the page can literally act as the light source for surrounding cards!

### 8.5. Axis 5: Nested `box-elevation` Behavior

When an elevated child (`.button { box-elevation: 4px }`) is placed inside an elevated parent (`.card { box-elevation: 8px }`), what happens?

| Option | Shadow Drop Distance onto Parent | 3D Height in Finite-Light Scene | Trade-Offs |
| :--- | :--- | :--- | :--- |
| **Option 5A: Local Drop + Cumulative 3D Height *(Baseline)*** | $4\text{px}$ | $8\text{px} + 4\text{px} = 12\text{px}$ | Physically exact: `.button` casts a `4px` shadow onto `.card`, and if `.card` rises toward a finite light bulb, `.button`'s shadow on `.card` subtly magnifies too. Requires summing ancestor elevations when finite lights are active. |
| **Option 5B: Strictly Local (Non-Accumulating)** | $4\text{px}$ | $4\text{px}$ | Simplest implementation: `.button`'s shadow depends *only* on its own `box-elevation: 4px` and its $(X, Y)$ position, completely ignoring ancestor `box-elevation` values. |
| **Option 5C: Absolute Z-Level (`z-elevation`)** | $12\text{px} - 8\text{px} = 4\text{px}$ *(child must declare `12px`)* | $12\text{px}$ | Matches Material Design's global `dp` elevation table (where an app bar is `4dp`, a card is `8dp`, and a button on the card must declare `12dp`). Poor component modularity: moving a button from the page (`0dp`) into a modal (`24dp`) breaks if its elevation is hardcoded as `4dp`. |

### 8.6. Axis 6: Naming (`light-*` vs. `box-shadow-*`)

- **`light-source` / `box-elevation` *(Baseline)***: Short, ergonomic, and leaves room for the same `light-source` context to potentially inform `text-elevation` or 3D surface filters in the future.
- **`box-shadow-light` / `box-shadow-elevation`**: Scopes the feature strictly under the existing `box-shadow-*` namespace in `css-borders-4`, making it immediately clear that this feature only generates box drop shadows and does not perform 3D Phong/PBR shading on element backgrounds.

---

## 9. Summary of Open Questions for CSSWG Discussion

1. **Coordinate Syntax (§8.1):** Should infinite and finite lights share a unified polar syntax (`<azimuth> <tilt> <distance>?`), use explicit functions (`directional(...)` vs. `point(...)`), or support a simple 2D slope vector?
2. **Composability with `box-shadow` (§8.2):** Should `box-elevation` automatically append shadow layers behind explicit `box-shadow` declarations, and should we *also* provide an `elevation(<length>)` function inside `box-shadow` for custom layer ordering?
3. **Scoping of Finite-Distance Lights (§8.4):** Should any element declaring `light-source` implicitly establish the reference box for finite-distance lights, or should `light-*` be ordinary inherited properties paired with an explicit scoping property?
4. **Nested Elevations (§8.5):** For finite-distance lights, is accumulating ancestor `box-elevation` into $Z_{\text{surface}}$ worth the implementation tracking, or is strictly local evaluation (Option 5B) sufficient?
5. **Optical Spread & Attenuation Formula (§4.2):** Should the umbra contraction (`spread`) and optional alpha falloff with height be fixed by the spec formula, or tunable via an optional parameter on the light source?
