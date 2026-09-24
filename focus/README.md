# Focus Navigation and Focusable Interface Explainer

## Problem

Web developers often need to inspect or programmatically manage focus order: for
example, when building custom focus traps, roving tabindexes, or accessibility
utilities. Currently, the platform provides no built-in way to query what
item would receive focus next (as if the user pressed `Tab` or `Shift+Tab`),
forcing authors to approximate browser focus-navigation heuristics in user
space.

Additionally, focus is not limited to regular DOM elements:
* Features like CSS carousels (`::scroll-marker`, `::scroll-button()`) introduce
  focusable pseudo-elements, while `CSSPseudoElement` instances are not yet
  universally exposed for all pseudo-element types.
* Focus may reside inside a shadow tree or opaque boundary where only the outer
  host element is accessible to the caller's scope.

Scripts need a unified representation of a focusable target to inspect active
focus, traverse sequential focus order, and programmatically move focus.

## Proposal

We propose introducing a `Focusable` interface, adding `activeFocusable` to
`DocumentOrShadowRoot`, and adding `nextFocusable()` and `previousFocusable()`
to traverse focus order:

```idl
[Exposed=Window]
interface Focusable {
  // Construct a Focusable from an Element (and optional pseudo-element
  // selector, e.g. "::scroll-button(inline-end)") or CSSPseudoElement.
  constructor((Element or CSSPseudoElement) target,
              optional CSSOMString? pseudoElement = null);

  // The shadow host (or container) when focus is inside a scope whose inner
  // target is not directly exposed to the caller; otherwise null.
  readonly attribute Element? host;

  // The focusable Element, or the originating Element when a pseudo-element
  // is focused (null if only `host` is exposed).
  readonly attribute Element? target;

  // The pseudo-element selector (e.g. "::scroll-marker") when this Focusable
  // represents a pseudo-element, avoiding a hard dependency on exposing
  // CSSPseudoElement instances; otherwise null.
  readonly attribute CSSOMString? pseudoElement;

  // Move focus to this target (whether an Element or pseudo-element).
  undefined focus(optional FocusOptions options = {});

  // Sequential focus navigation starting from this Focusable.
  Focusable? nextFocusable();
  Focusable? previousFocusable();
};

partial interface mixin DocumentOrShadowRoot {
  readonly attribute Focusable? activeFocusable;
};

partial interface Element {
  Focusable? nextFocusable();
  Focusable? previousFocusable();
};

partial interface CSSPseudoElement {
  undefined focus(optional FocusOptions options = {});
  Focusable? nextFocusable();
  Focusable? previousFocusable();
};
```

### Converting Between `Element`, `CSSPseudoElement`, and `Focusable`

Both `Element` and `CSSPseudoElement` expose `nextFocusable()` and
`previousFocusable()` directly, and `CSSPseudoElement` also exposes `focus()`
for parity with `HTMLElement.focus()`.

In addition, `Focusable` provides `focus()`, `nextFocusable()`, and
`previousFocusable()` directly on the descriptor object:

* **Why `Focusable` in addition to `CSSPseudoElement`?** Putting `focus()` and
  navigation methods on `Focusable` (along with the
  `new Focusable(element, "::scroll-marker")` constructor) lets authors
  programmatically focus and traverse pseudo-elements using their originating
  `Element` and selector string, even when `CSSPseudoElement` instances are not
  exposed. When `CSSPseudoElement` is available, `pseudo.focus()`,
  `pseudo.nextFocusable()`, and `new Focusable(pseudo)` work seamlessly as well.
* **How to obtain a `Focusable`?**
  1. **From active focus**: `document.activeFocusable`.
  2. **By construction**: `new Focusable(element)`,
     `new Focusable(element, "::scroll-button(inline-end)")`, or
     `new Focusable(pseudoElement)`.
  3. **By traversal**: calling `nextFocusable()` or `previousFocusable()` on any
     `Element`, `CSSPseudoElement`, or `Focusable`.

### `document.activeFocusable`

Today, `document.activeElement` always returns an `Element`. When a focusable
pseudo-element holds focus, `document.activeElement` points to its originating
`Element` for backwards compatibility.

`document.activeFocusable` returns a `Focusable` describing the currently
focused item:
* **Regular element focused**: `target` is the focused `Element`,
  `pseudoElement` is `null`, and `host` is `null`.
* **Pseudo-element focused**: `target` is the originating `Element`, and
  `pseudoElement` is its selector string (e.g., `"::scroll-marker"`).
* **Unexposed inner focus (e.g. closed shadow tree)**: `host` is the outermost
  accessible host `Element`, while `target` is `null`.

### Focus Navigation Behavior and Caveats

* **Sequential focus order**: `nextFocusable()` and `previousFocusable()` follow
  the browser's sequential focus navigation order (`Tab` / `Shift+Tab`) starting
  from the receiver (`Element` or `Focusable`), even if the receiver is not
  currently focused (or not focusable itself, such as
  `document.documentElement`).
* **Boundary return value**: When called on the last focusable item in the
  document, `nextFocusable()` returns `null`. Symmetrically,
  `previousFocusable()` returns `null` when there is no prior focusable item.
* **Snapshot of current page state**: Because moving focus (or running script in
  between steps) can mutate the DOM or trigger style changes (e.g.,
  `:focus-within` revealing new elements, or `focus` event listeners modifying
  attributes), the returned `Focusable` is only guaranteed to be valid **as of
  the current state of the page** when the function is called.

## Examples

### Advancing focus from the currently focused item

```js
const next = document.activeFocusable?.nextFocusable();
if (next) {
  next.focus();
}
```

### Focusing a specific pseudo-element directly

```js
const buttonFocusable = new Focusable(carousel, "::scroll-button(inline-end)");
buttonFocusable.focus();
```

### Collecting all focusable items on the page

Starting from `document.documentElement` and advancing until `nextFocusable()`
returns `null` yields all currently focusable items in tab order:

```js
function getAllFocusables() {
  const focusables = [];
  let curr = document.documentElement.nextFocusable();
  while (curr) {
    focusables.push(curr);
    curr = curr.nextFocusable();
  }
  return focusables;
}
```

## Alternatives Considered

### Returning the full list of focusable elements

We considered providing an API that directly returns the complete list of
focusable items (e.g., `document.getFocusables()`). We decided against this for
two main reasons:

1. **Performance**: Computing focusability requires layout and style checks
   across the entire DOM tree. Many use cases only need the immediately adjacent
   focusable item; computing the full list upfront does unnecessary work on
   large pages.
2. **Staleness**: Because focusing an item can change styles and DOM structure,
   a precomputed list can immediately become stale as the page is traversed.

By exposing `nextFocusable()` and `previousFocusable()` as primitives,
single-step lookups remain fast, and developers who genuinely need the full list
can trivially construct it in user space (as shown in the example above).
