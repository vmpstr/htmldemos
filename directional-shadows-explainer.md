# CSS Directional Lighting & Box Elevation (`light-source` & `box-elevation`)

## Problem

In most design systems (Material, HIG, Fluent, Tailwind, etc.), shadows are used to indicate an element's elevation above its surface under a shared lighting environment.

In CSS today, authors have to translate that mental model into low-level `box-shadow` offsets, blur radii, and spreads by hand for every elevation step:

```css
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

This comes with a few recurring problems:

1. **Hard to maintain:** Changing the light angle or softness means recalculating every shadow token across the stylesheet.
2. **Clobbers `box-shadow` outlines and rings:** `box-shadow` is commonly used for focus rings (`0 0 0 2px <color>`) or hairline borders. Adding a focus ring overwrites the element's elevation shadow unless everything is wired through custom properties.
3. **No spatial awareness for local lights:** Making shadows diverge from a local light source across a grid of cards—or shift as elements scroll past a light—requires JavaScript `scroll` and `ResizeObserver` listeners updating inline styles every frame.
4. **Manual inset conversion:** Making a recessed element (like an input well) match the same light source requires manually switching to `inset` and re-tuning the offsets and blur.

## Proposal

We propose splitting lighting into two parts:
1. A container defines one or more light sources using `light-source` (and its longhands).
2. Descendant elements declare their height relative to their surface using `box-elevation`.

The browser then automatically generates the corresponding shadow layers using the existing `box-shadow` rendering pipeline (so `border-radius`, `corner-shape`, `border-shape`, and `box-decoration-break` all work as expected).

### Defining a Light Source

`light-source` is a shorthand for five coordinating list longhands:
- `light-direction`: `<azimuth> <tilt>? <distance>?`
- `light-origin`: `<position>` (default `50% 50%`)
- `light-softness`: `<percentage> | <number> | <angle> | <length>` (default `50%`)
- `light-color`: `<color>` (default `rgb(0 0 0 / 20%)`)
- `light-attachment`: `local | scrollport | viewport` (default `local`)

Multiple lights can be specified as a comma-separated list, just like `box-shadow` or `background`. Each light produces one shadow layer on elevated descendants.

```
<light-layer> = <'light-direction'>
                && [ / <'light-softness'> ]?
                && <'light-color'>?
                && [ at <'light-origin'> ]?
                && <'light-attachment'>?
```
*(Note: The `/` before `light-softness` disambiguates it from `<distance>` and `<tilt>` when both use `<length>` or `<angle>` units.)*

#### `light-direction`

```
Value: [ none | <azimuth> <tilt>? <distance>? ]#
Initial: none
```

Where:
- **`<azimuth>`** (`<angle> | [ left | right ] || [ top | bottom ]`): The compass direction on the page **from which** the light shines.
  - `0deg` (`top`) shines from the top edge, casting shadows downward.
  - `90deg` (`right`) shines from the right edge, casting shadows leftward.
  - `180deg` (`bottom`) shines from the bottom edge, casting shadows upward.
  - `270deg` (`left`) shines from the left edge, casting shadows rightward.
- **`<tilt>`** (`<angle [0deg, 90deg)>`, default `0deg`): How far the light is tilted away from straight overhead. `0deg` is directly overhead (producing a centered ambient shadow for distant lights); `45deg` casts a shadow whose offset equals the element's `box-elevation`.
- **`<distance>`** (`infinite | <length [0px, ∞]>`, default `infinite`): How far away the light source is from `light-origin`.
  - When `infinite`, light rays are parallel everywhere (a directional light).
  - When a finite `<length>` is given (e.g. `600px`), the light sits at that 3D distance from `light-origin`, and shadows diverge based on each element's position relative to the light.

#### `light-origin`

```
Value: <position>#
Initial: 50% 50%
```

Sets the anchor point on the lighting container's border box from which finite-distance lights are measured (similar to `perspective-origin`). When a light's distance is `infinite`, `light-origin` has no effect.

#### `light-softness`

```
Value: [ <percentage [0,∞]> | <number [0,∞]> | <angle [0deg, 90deg)> | <length [0,∞]> ]#
Initial: 50%
```

Controls how quickly the shadow's blur radius grows with `box-elevation`:
- **`<percentage>` or `<number>`**: The blur-to-elevation ratio (`100%` = `1`).
  - `0%`: Crisp shadow (`0px` blur at any elevation).
  - `50%`: `0.5px` of blur per `1px` of elevation (e.g. `5px` blur at `box-elevation: 10px`).
  - `100%`: `1px` of blur per `1px` of elevation.
- **`<angle>`**: The angular radius of the light source.
- **`<length>`**: The physical radius of a finite-distance light source (if distance is `infinite`, a `<length>` softness is treated as `0`).

#### `light-color`

```
Value: <color>#
Initial: rgb(0 0 0 / 20%)
```

Sets the base shadow color for each light source.

ISSUE: Should this be currentColor? How are shadows typically represented in dark mode?

#### `light-attachment`

```
Value: [ local | scrollport | viewport ]#
Initial: local
```

Controls how finite-distance lights behave when scrolling:
- **`local` (default):** The light is attached to the container's scrollable content. Scrolling moves the content and the light together, so shadows remain static during scroll.
- **`scrollport`:** The light is attached to the container's scrollport (border box). As content scrolls inside the container, elements move past the light and their shadows shift dynamically.
- **`viewport`:** The light is fixed to the viewport. Scrolling any ancestor moves elements relative to the light.

### Elevating Elements: `box-elevation`

```
Name: box-elevation
Value: <length>
Initial: 0px
Inherited: no
```

- **`0px` (default):** The element sits flat on its surface and casts no elevation shadow.
- **Positive `<length>` (e.g. `box-elevation: 8px`):** The element is raised above its surface, generating one `outset` shadow layer per light in the nearest ancestor with `light-source`.
- **Negative `<length>` (e.g. `box-elevation: -4px`):** The element is recessed into its surface, generating one `inset` shadow layer per light.

**Composition with `box-shadow`:** `box-elevation` does not overwrite `box-shadow`. If an element has both `box-elevation` and `box-shadow`, explicit `box-shadow` layers are painted in front and `box-elevation` shadow layers are painted immediately behind them. Setting `box-elevation` does not create a stacking context or containing block.

## Examples

### 1. Global Key + Ambient Light Setup

A typical design system setup defines a directional key light and an overhead ambient light once on `:root`. Components only set `box-elevation`, leaving `box-shadow` free for focus rings:

```css
:root {
  /* Key light from top-left (330deg) tilted 25deg, plus an overhead ambient light */
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
  box-elevation: 16px;
}

/* Focus ring doesn't clobber the elevation shadow */
.card:focus-visible {
  box-shadow: 0 0 0 3px oklch(65% 0.2 250);
}

/* Sunken input well inside the card */
.card input {
  box-elevation: -3px;
}
```

### 2. Positional Light on a Grid (Static on Scroll)

With a finite distance (`600px`), shadows diverge away from `light-origin`. Because `light-attachment` defaults to `local`, scrolling the page does not cause the shadows to shift:

```css
.bento-grid {
  light-source: 0deg 0deg 600px / 120px rgb(0 0 0 / 20%) at 50% 0%;
}

.bento-card {
  box-elevation: 8px;
}

.bento-card .action-button {
  box-elevation: 4px;
}
```

### 3. Scroll-Reactive Light (`scrollport`)

Opting into `scrollport` pins the light to the scroll container's border box so shadows shift as items scroll past it:

```css
.scrolling-gallery {
  overflow-y: auto;
  height: 500px;
  light-source:
    0deg 0deg 450px / 90px rgb(0 0 0 / 22%) at 50% 20% scrollport,
    0deg 0deg / 100% rgb(0 0 0 / 8%);
}

.gallery-item {
  box-elevation: 10px;
}
```

## Alternatives and Open Questions

### 1. Specifying Light Position (`light-direction` vs. functions vs. 3D coordinates)

The proposal uses polar angles with an optional distance (`<azimuth> <tilt>? <distance>?`) paired with `light-origin`. This unifies directional and positional lights (`infinite` is just the default distance), and polar angles are natural for directional UI shadows.

Other options we considered:
- **Explicit `directional()` and `point()` functions:** e.g. `light-position: directional(330deg 25deg)` vs. `light-position: point(50% 0% 600px)`. This mirrors SVG's `<feDistantLight>` / `<fePointLight>` and folds `light-origin` into `point()`, at the cost of making interpolation between the two modes awkward.
- **Cartesian 3D coordinates only (`<position> <length>`):** e.g. `light-position: 50% -1000px 2000px`. Simple mental model, but cannot express a true container-size-independent directional light without huge magic numbers.
- **2D slope multipliers:** e.g. `light-slope: 0.25 0.5` (offset per `1px` of elevation). Easy to reason about in pixels, but doesn't extend to finite-distance point lights.

### 2. Connecting Elevation to `box-shadow`

Having `box-elevation` automatically paint behind explicit `box-shadow` layers avoids the common problem where adding a focus ring wipes out the elevation shadow. However, it means `box-shadow: none` won't clear a `box-elevation` shadow (you have to set `box-elevation: 0px`), and you can't place a manual `box-shadow` layer *behind* the elevation shadow.

Alternatives:
- **Add an `elevation(<length>)` function inside `box-shadow`:** e.g. `box-shadow: 0 0 0 2px royalblue, elevation(4px)`. This could also be supported alongside `box-elevation` for cases where custom layer ordering is needed.
- **`box-shadow: auto` keyword:** `box-elevation` only takes effect when `box-shadow` includes `auto`. This is more explicit, but reintroduces the footgun where `box-shadow: 0 0 0 2px blue` clobbers the shadow unless the author remembers to write `, auto`.
- **Reuse `translateZ()` / `translate`:** Avoids a new elevation property, but transforms create a stacking context, capture `position: fixed` descendants, and visibly scale the element when `perspective` is present.

### 3. Multi-Light Lists vs. Dedicated `light-ambient`

A single directional shadow often looks harsh without a soft overhead ambient shadow. The proposal handles this by allowing a comma-separated list of lights in `light-source`. An alternative (similar to Android's `RenderNode` lighting) is a single `light-source` paired with a dedicated `light-ambient: <softness> <color>` property. Since an ambient light is already expressible as a `0deg 0deg` light layer, a list seems more flexible without adding another property.

### 4. Container Lookup vs. Inheritance

In the proposal, `light-source` is non-inherited and any element with `light-source != none` acts as the lighting container for its descendants.

Alternatively, `light-*` could be standard inherited properties. For infinite-distance lights, no container box is needed at all; for finite-distance lights, `light-origin` could resolve against the nearest ancestor with an explicit scoping property (or the viewport by default), or even reference an anchor element via `anchor(--lamp center)`.

### 5. Nested `box-elevation`

When an elevated child (e.g. `.button { box-elevation: 4px }`) sits inside an elevated parent (`.card { box-elevation: 8px }`), the button casts a `4px` shadow onto the card.

For directional (`infinite` distance) lights, only the child's own `4px` elevation matters. For finite-distance lights, there is an open question of whether the child's 3D position should accumulate its ancestors' elevations (`8px + 4px = 12px`, bringing it slightly closer to the light) or be evaluated strictly locally (`4px`) to avoid tracking cumulative elevation up the tree.

### 6. Naming

We used `light-source` and `box-elevation` for brevity, and because `light-source` could potentially inform other properties like `text-elevation` in the future. If we want to scope this strictly to box shadows, `box-shadow-light` and `box-shadow-elevation` are an option.

---

## Appendix: Mathematical Model & Implementation Details

This appendix describes one way to map `light-source` and `box-elevation` onto standard `box-shadow` parameters (`position`, `offset-x`, `offset-y`, `blur`, and `spread`).

### A.1. Lighting Container & Nested Elevations

An element $E$ with `box-elevation` $h \neq 0$ resolves its lighting against the nearest ancestor $C$ with `light-direction != none` (its lighting container).

Because $E$'s shadow is painted immediately behind $E$, the shadow lands on $E$'s parent surface, so the drop distance that the shadow travels is always $E$'s own local elevation $|h|$.

For finite-distance lights, if intermediate ancestors between $C$ and $E$ also have `box-elevation`, we can optionally account for their cumulative height when computing how close $E$ is to the light:
- Let $Z_{\text{surface}}$ be the sum of `box-elevation` on ancestors between $C$ and $E$ (or `0px` if evaluated strictly locally).
- If $h > 0$ (raised element), the top of $E$ casts the shadow onto $Z_{\text{surface}}$, so the shadow-casting edge is at $Z_{\text{caster}} = Z_{\text{surface}} + h$.
- If $h < 0$ (recessed element), the rim at $Z_{\text{surface}}$ casts an inset shadow onto the recessed floor at $Z_{\text{surface}} + h$, so the shadow-casting edge is at $Z_{\text{caster}} = Z_{\text{surface}}$.

### A.2. Converting a Light into `box-shadow` Parameters

For each light layer on $C$ with azimuth $\alpha$, tilt $\phi \in [0^\circ, 90^\circ)$, distance $d$, softness $\kappa$, and color $c$:

- **Shadow position (`outset` vs. `inset`):**
  - If $h > 0$, the shadow is `outset`.
  - If $h < 0$, the shadow is `inset`.

*(Note: In CSS `box-shadow`, positive `(offset-x, offset-y)` values on an `inset` shadow shift the opening down and right, which places the visible inner shadow on the top and left edges. This means the exact same offset formulas work for both `outset` and `inset` shadows.)*

#### Case 1: Infinite Distance (`d = infinite`)

When distance is `infinite`, light rays are parallel across the container:

$$
\text{offset}_x = -|h| \cdot \tan(\phi) \cdot \sin(\alpha)
$$

$$
\text{offset}_y = +|h| \cdot \tan(\phi) \cdot \cos(\alpha)
$$

$$
\text{blur} = |h| \cdot \kappa
$$

where $\kappa$ is the softness value as a number (`100%` $\to 1$), or $\kappa = \tan(\delta)$ when `light-softness` is specified as an `<angle>` $\delta$.

To keep very soft shadows from bleeding out behind the light-facing edge of the box, the umbra can optionally be contracted slightly via negative spread:

$$
\text{spread} = -|h| \cdot \kappa \cdot 0.25
$$

ISSUE: Should umbra contraction (`spread`) and any alpha falloff with elevation be fixed by the spec formula, or tunable via an optional parameter?

#### Case 2: Finite Distance (`d = <length>`)

Let $(X_0, Y_0)$ be the resolved `light-origin` on $C$. The light's 3D position relative to $C$ is:

$$
X_L = X_0 + d \cdot \sin(\phi)\sin(\alpha)
$$

$$
Y_L = Y_0 - d \cdot \sin(\phi)\cos(\alpha)
$$

$$
Z_L = d \cdot \cos(\phi)
$$

Let $(X_E, Y_E)$ be the center of element $E$'s border box in $C$'s coordinate space, and let $\Delta Z = Z_L - Z_{\text{caster}}$ (clamped to be positive) be the vertical distance from the light to the casting plane.

By similar triangles, the projection factor is:

$$
k = \frac{|h|}{\Delta Z}
$$

Which gives:

$$
\text{offset}_x = (X_E - X_L) \cdot k
$$

$$
\text{offset}_y = (Y_E - Y_L) \cdot k
$$

For the blur radius:
- If `light-softness` is a `<length>` $R_L$, then $\text{blur} = R_L \cdot k$.
- If `light-softness` is a ratio $\kappa$, the effective radius at height $Z_L$ is $R_L = \kappa \cdot Z_L$, so $\text{blur} = \kappa \cdot Z_L \cdot k$.

As $d \to \infty$, these finite-distance equations converge to the infinite-distance equations in Case 1.

### A.3. Coordinate Spaces for `light-attachment`

For finite-distance lights, $(X_0, Y_0)$ and $(X_E, Y_E)$ are resolved depending on `light-attachment`:

- **`local`:** $(X_0, Y_0)$ is resolved against $C$'s border-box size at $C$'s content origin. $(X_E, Y_E)$ uses $E$'s layout offset relative to $C$, ignoring scroll offsets of $C$ or intermediate scrollers. Shadows can be computed at layout time and do not change during compositor scrolling.
- **`scrollport`:** $(X_0, Y_0)$ is anchored to $C$'s border box (the scrollport). $(X_E, Y_E)$ includes scroll offsets between $C$ and $E$, so shadows update as content scrolls (similar to `perspective-origin`).
- **`viewport`:** $(X_0, Y_0)$ is resolved against the viewport, and $(X_E, Y_E)$ is $E$'s position relative to the viewport (similar to `background-attachment: fixed`).
