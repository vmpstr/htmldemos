# Focus Navigation and Pseudo-Element Focus Explainer

## Problem

Web developers often need to inspect or programmatically manage focus order: for
example, when building custom focus traps, roving tabindexes, or accessibility
utilities. Currently, the platform provides no built-in way to query what
element would receive focus next (as if the user pressed `Tab` or `Shift+Tab`),
forcing authors to approximate browser focus-navigation heuristics in user
space.

Additionally, with features like CSS carousels (`::scroll-marker`,
`::scroll-button()`) introducing focusable pseudo-elements, scripts need a way
to both discover these pseudo-elements in the focus order and programmatically
move focus to them.

## Proposal

We propose two additions to the platform:

1. **Extend `CSSPseudoElement` with `focus()`**, matching
   `HTMLElement.focus(optional FocusOptions options)`.
2. **Add `nextFocusable()` and `previousFocusable()` to both `Element` and
   `CSSPseudoElement`**, returning the next or previous focusable `Element` or
   `CSSPseudoElement` in sequential focus navigation order (as if pressing `Tab`
   or `Shift+Tab` from the receiver), or `null` if none exists.

```idl
typedef (Element or CSSPseudoElement) Focusable;

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

### Behavior and Caveats

* **Sequential focus order**: `nextFocusable()` and `previousFocusable()` follow
  the browser's sequential focus navigation order (`Tab` / `Shift+Tab`) starting
  from the element or pseudo-element on which the method is called, even if the
  receiver is not currently focused (or not focusable itself, such as
  `document.documentElement`).
* **Boundary return value**: When called on the last focusable item in the
  document, `nextFocusable()` returns `null`. Symmetrically,
  `previousFocusable()` returns `null` when there is no prior focusable item.
* **Snapshot of current page state**: Because moving focus (or running script in
  between steps) can mutate the DOM or trigger style changes (e.g.,
  `:focus-within` revealing new elements, or `focus` event listeners modifying
  attributes), the returned value is only guaranteed to be valid **as of the
  current state of the page** when the function is called.

## Examples

### Focusing the next item

```js
const next = activeItem.nextFocusable();
if (next) {
  next.focus();
}
```

### Collecting all focusable items on the page

Starting from `document.documentElement` and advancing until `nextFocusable()`
returns `null` yields all currently focusable elements and pseudo-elements in
tab order:

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
focusable items (e.g., `document.getFocusableElements()`). We decided against
this for two main reasons:

1. **Performance**: Computing focusability requires layout and style checks
   across the entire DOM tree. Many use cases only need the immediately adjacent
   focusable item; computing the full list upfront does unnecessary work on
   large pages.
2. **Staleness**: Because focusing an element can change styles and DOM
   structure, a precomputed list can immediately become stale as the page is
   traversed.

By exposing `nextFocusable()` and `previousFocusable()` as primitives,
single-step lookups remain fast, and developers who genuinely need the full list
can trivially construct it in user space (as shown in the example above).
