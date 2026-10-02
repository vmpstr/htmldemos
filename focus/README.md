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
  focusable pseudo-elements.
* Focus may reside inside a shadow tree or opaque boundary where only the outer
  host element is accessible to the caller's scope.

Script needs a unified representation of a focusable target to inspect active
focus, traverse sequential focus order, and programmatically move focus.

## Use Cases

### 1. Focus Traps and Modal Containers

When building custom dialogs, popovers, or menus that trap focus, authors often
rely on `querySelectorAll` with a list of selectors (`a[href]`,
`button:not([disabled])`, `[tabindex]:not([tabindex="-1"])`, etc.). This
approach can miss several cases:

* Focusable CSS pseudo-elements (`::scroll-marker`, `::scroll-button()`).
* Non-DOM focus order defined by CSS `reading-flow`.
* Shadow hosts with internal focusable elements.
* Elements affected by `inert`, `content-visibility: hidden`, disabled
  `<fieldset>` ancestors, or layout state.

Generally, maintaining a `querySelectorAll` driven implementation can fall
behind as new features are implemented. With `nextFocusable()` and
`previousFocusable()`, a focus trap can query the browser's sequential focus
order directly.

### 2. Saving and Restoring Focus

When a dialog or menu closes, applications often restore focus to the previously
focused item. Saving `document.activeElement` loses precision when a
pseudo-element is focused, because `activeElement` only points to the
originating element. Saving `document.activeFocusable` and later calling
`focus()` on it restores focus to the exact element or pseudo-element.

### 3. Programmatic Control and Testing of Pseudo-Elements

CSS carousels introduce focusable `::scroll-marker` and `::scroll-button()`
pseudo-elements that participate in `Tab` order. Authors and testing tools need
to perform several operations on these pseudo-elements:

* Check whether a pseudo-element currently holds focus (`document.activeFocusable`).
* Programmatically focus a specific pseudo-element
  (`carousel.pseudo("::scroll-button(inline-end)")?.focus()`).
* Step forward or backward from a pseudo-element in focus order.

### 4. Custom Keyboard Navigation

Widgets such as toolbars, menus, and data grids often move focus to the next or
previous focusable item in response to arrow keys or custom shortcuts.
`nextFocusable()` and `previousFocusable()` let script use the browser's focus
order without reimplementing `tabindex` or `reading-flow` logic.

### 5. Skip Links and Section Jumps

Helpers that move focus to the first focusable control after a non-focusable
heading can call `heading.nextFocusable()?.focus()` directly, without making the
heading itself focusable or walking the DOM tree manually.

## Proposal

We propose introducing a `Focusable` interface, adding `activeFocusable` to
`DocumentOrShadowRoot`, and adding `nextFocusable()` and `previousFocusable()`
to traverse focus order:

```idl
[Exposed=Window]
interface Focusable {
  // Construct a Focusable from an Element or CSSPseudoElement.
  constructor((Element or CSSPseudoElement) target);

  // The shadow host (or container) when focus is inside a shadow scope;
  // otherwise null.
  readonly attribute Element? shadowHost;

  // The focusable Element, or the originating Element when a pseudo-element
  // is focused (null if only `shadowHost` is exposed).
  readonly attribute Element? target;

  // The focusable CSSPseudoElement, if the focus is on a pseudo-element.
  // Otherwise, null.
  readonly attribute CSSPseudoElement? pseudoElement;

  // Move focus to this target (whether an Element, pseudo-element, or host).
  undefined focus(optional FocusOptions options = {});

  // Sequential focus navigation starting from this Focusable.
  Focusable? nextFocusable();
  Focusable? previousFocusable();
};

partial interface mixin DocumentOrShadowRoot {
  readonly attribute Focusable? activeFocusable;
};

// Convenience enhancements to avoid requiring creating a new Focusable.
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

* **Why `Focusable` in addition to `Element` and `CSSPseudoElement`?** A focus
  target may be a regular `Element`, a `CSSPseudoElement` (which also has an
  originating `target` `Element`), or an item inside a shadow tree (represented
  by its `shadowHost`). `Focusable` provides a single unified type for all of
  these cases, and putting `focus()`, `nextFocusable()`, and
  `previousFocusable()` directly on `Focusable` lets authors traverse and move
  focus without branching on what kind of item is focused.
* **How to obtain a `Focusable`?**
  1. **From active focus**: `document.activeFocusable`.
  2. **By construction**: `new Focusable(element)` or
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
  `pseudoElement` is `null`, and `shadowHost` is `null`.
* **Pseudo-element focused**: `target` is the originating `Element`,
  `pseudoElement` is the focused `CSSPseudoElement`, and `shadowHost` is `null`.
* **Focus inside a shadow tree (or slotted into one)**: `shadowHost` is the
  outermost shadow host `Element` in the caller's scope, while `target` is
  `null` and `pseudoElement` is `null`.
* **No focus within the document**: `activeFocusable` is null when the document
  doesn't have any focused item within it.

### Focus Navigation Behavior and Caveats

* **Sequential focus order**: `nextFocusable()` and `previousFocusable()` follow
  the browser's sequential focus navigation order (`Tab` / `Shift+Tab`,
  including `tabindex` ordering and CSS `reading-flow`) starting from the
  receiver (`Element`, `CSSPseudoElement`, or `Focusable`), even if the receiver
  is not currently focused or is not focusable itself (such as
  `document.documentElement`). Elements that do not participate in sequential
  focus navigation (such as those with `tabindex="-1"`, `disabled`, `inert`, or
  `display: none`) are skipped. This follows [Sequential Focus
  Navigation](https://html.spec.whatwg.org/multipage/interaction.html#sequential-focus-navigation) and [reading-flow](https://drafts.csswg.org/css-display-4/#reading-flow).

Consider the following DOM:
```html
<div id="container">
  <button id="default">Default</button>
  <button id="neg" tabindex="-1">Skipped</button>
  <button id="second" tabindex="2">Second</button>
  <button id="disabled" disabled>Disabled</button>
  <button id="first" tabindex="1">First</button>
</div>
```

Here is the chain of `Focusable`s as would be returned by repeated calls to
`nextFocusable()` starting from `document.documentElement`:
```
Focusable { shadowHost: null, target: <button#first>, pseudoElement: null }
  |
  v
Focusable { shadowHost: null, target: <button#second>, pseudoElement: null }
  |
  v
Focusable { shadowHost: null, target: <button#default>, pseudoElement: null }
  |
  v
null
```

Similarly, CSS `reading-flow` can reorder focus navigation:
```html
<div style="display: flex; reading-flow: flex-visual">
  <button id="one" style="order: 3">One</button>
  <button id="two" style="order: 1">Two</button>
  <button id="three" style="order: 2">Three</button>
</div>
```

Here is the chain of `Focusable`s as would be returned by repeated calls to
`nextFocusable()` starting from `document.documentElement`:
```
Focusable { shadowHost: null, target: <button#two>, pseudoElement: null }
  |
  v
Focusable { shadowHost: null, target: <button#three>, pseudoElement: null }
  |
  v
Focusable { shadowHost: null, target: <button#one>, pseudoElement: null }
  |
  v
null
```

* **Focusable pseudo-elements**: Focusable pseudo-elements (such as
  `::scroll-button()` and `::scroll-marker` on CSS carousels) are visited in the
  same order as `Tab` and `Shift+Tab`. When a pseudo-element is visited,
  `target` is set to its originating `Element` and `pseudoElement` is set to the
  `CSSPseudoElement`.

Consider a CSS carousel with scroll buttons and a scroll marker group:
```html
<style>
  #carousel {
    overflow: auto;
    scroll-marker-group: before;
  }
  #carousel::scroll-button(inline-start) { content: "<"; }
  #carousel::scroll-button(inline-end) { content: ">"; }
  .slide::scroll-marker { content: ""; }
</style>

<div id="carousel">
  <div id="slide-1" class="slide">
    <a id="slide-link" href="#">Link</a>
  </div>
</div>
```

Here is the chain of `Focusable`s as would be returned by repeated calls to
`nextFocusable()` starting from `document.documentElement`:
```
Focusable { shadowHost: null, target: <div#slide-1>, pseudoElement: <div#slide-1>::scroll-marker }
  |
  v
Focusable { shadowHost: null, target: <div#carousel>, pseudoElement: <div#carousel>::scroll-button(inline-start) }
  |
  v
Focusable { shadowHost: null, target: <div#carousel>, pseudoElement: <div#carousel>::scroll-button(inline-end) }
  |
  v
Focusable { shadowHost: null, target: <a#slide-link>, pseudoElement: null }
  |
  v
null
```

* **Shadow trees and slotted light DOM as a single entry**: When traversing
  across a shadow host, the shadow tree and any light DOM slotted into it are
  treated as a single entry in the `nextFocusable()` / `previousFocusable()`
  chain, assuming there is anything focusable inside:
  * Stepping onto the shadow host yields `{ shadowHost: host, target: null, pseudoElement: null }`.
  * Calling `nextFocusable()` on that `Focusable` advances past the entire
    shadow tree (and its slotted light DOM) to the next focusable item after the
    shadow host, rather than repeatedly returning the same shadow host.
  * If nothing inside the shadow tree or its slotted light DOM is focusable (and
    the host itself is not focusable), the shadow host is skipped.

Consider a shadow host with internal focusable elements and a slotted light DOM
link:
```html
<button id="before">Before</button>
<my-widget>
  #shadow-root
    <button id="inner-1">Inner 1</button>
    <slot></slot>
    <button id="inner-2">Inner 2</button>
  <a id="slotted" href="#">Slotted</a>
</my-widget>
<button id="after">After</button>
```

Here is the chain of `Focusable`s as would be returned by repeated calls to
`nextFocusable()` starting from `document.documentElement`:
```
Focusable { shadowHost: null, target: <button#before>, pseudoElement: null }
  |
  v
Focusable { shadowHost: <my-widget>, target: null, pseudoElement: null }
  |
  v
Focusable { shadowHost: null, target: <button#after>, pseudoElement: null }
  |
  v
null
```

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
carousel.pseudo("::scroll-button(inline-end)")?.focus();
```

### Saving and restoring focus (including pseudo-elements)

```js
const previouslyFocused = document.activeFocusable;

openModalDialog();
// ... later, when the modal closes:
previouslyFocused?.focus();
```

### Wrapping focus inside a custom container (Focus Trap)

```js
function handleTabInContainer(event, container) {
  if (event.key !== "Tab") return;

  const current = document.activeFocusable;
  const next = event.shiftKey
    ? current?.previousFocusable()
    : current?.nextFocusable();

  const nextNode = next?.target ?? next?.shadowHost;
  if (!nextNode || !container.contains(nextNode)) {
    event.preventDefault();
    if (event.shiftKey) {
      let item = container.nextFocusable();
      let last = null;
      while (item && container.contains(item.target ?? item.shadowHost)) {
        last = item;
        item = item.nextFocusable();
      }
      last?.focus();
    } else {
      const first = container.nextFocusable();
      if (first && container.contains(first.target ?? first.shadowHost)) {
        first.focus();
      }
    }
  }
}
```

### Collecting all focusable items on the page

Starting from `document.documentElement` and advancing until `nextFocusable()`
returns `null` yields all currently focusable items in tab order (with each
focusable shadow host represented as a single entry):

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

## Opt-in Shadow Tree Inspection and Traversal

> **Note:** This section describes an optional extension for exposing focusable
> items inside known `ShadowRoot`s. The rest of the proposal above remains
> useful on its own even without this extension.

By default, a shadow host (including its shadow tree and slotted light DOM) is
treated as a single entry with `shadowHost` set and `target: null`. Following
the example of `Element.prototype.getHTML({ shadowRoots })`, if a set of
`ShadowRoot`s is provided when constructing a `Focusable`, we can expose the
focusable elements inside those shadow trees alongside the shadow host.

### Discussions: Focus Within Custom Elements

The existing `Focusable` constructor gains an optional `shadowRoots` parameter,
and a new constructor accepts an existing `Focusable` and `shadowRoots`:

```idl
partial interface Focusable {
  constructor((Element or CSSPseudoElement) target,
              optional sequence<ShadowRoot> shadowRoots = []);

  constructor(Focusable focusable,
              optional sequence<ShadowRoot> shadowRoots = []);
};
```

### Behavior with `shadowRoots`

* **Exposing `target` alongside `shadowHost`**: When a shadow host's
  `ShadowRoot` is included in `shadowRoots`, focusable items inside that shadow
  tree (or slotted into it) populate `target` (and `pseudoElement`, if
  applicable) alongside `shadowHost`.
* **Constructing from an existing `Focusable`**: `new Focusable(focusable, shadowRoots)`
  creates a new `Focusable` with the given `shadowRoots` replacing any previous
  set (_perhaps we want to append to the set?_):
  * If the initial `Focusable` was a shadow host with `target: null` that
    originated from a specific inner focus target (such as
    `document.activeFocusable` when an element inside that shadow root is
    focused), providing the matching `ShadowRoot` populates `target`.
  * If the initial `Focusable` was obtained by stepping onto a shadow host as a
    single entry during traversal (so it does not refer to a specific inner
    element), or if the inner target is inside a nested `ShadowRoot` that was
    not provided, `target` may be `null`.
* **Traversal**: `nextFocusable()` and `previousFocusable()` propagate the
  receiver's `shadowRoots` to the returned `Focusable` and step through the
  focusable items inside those shadow roots instead of treating the host as a
  single entry.

_Issue: is document.activeFocusable always shadowRoot-free?_

Using the same `<my-widget>` DOM from earlier, here is the chain of `Focusable`s
as would be returned by repeated calls to `nextFocusable()` starting from
`new Focusable(document.documentElement, [myWidget.shadowRoot])`:
```
Focusable { shadowHost: null, target: <button#before>, pseudoElement: null }
  |
  v
Focusable { shadowHost: <my-widget>, target: <button#inner-1>, pseudoElement: null }
  |
  v
Focusable { shadowHost: <my-widget>, target: <a#slotted>, pseudoElement: null }
  |
  v
Focusable { shadowHost: <my-widget>, target: <button#inner-2>, pseudoElement: null }
  |
  v
Focusable { shadowHost: null, target: <button#after>, pseudoElement: null }
  |
  v
null
```

### Shadow Root Examples

#### Inspecting active focus inside a known shadow root

```js
// Without shadowRoots, activeFocusable only exposes the shadowHost:
const active = document.activeFocusable;
// active.shadowHost === myWidget, active.target === null

// Providing the ShadowRoot populates target alongside shadowHost:
const detailed = new Focusable(active, [myWidget.shadowRoot]);
// detailed.shadowHost === myWidget, detailed.target === innerButton
```

#### Traversing focusable items across known shadow roots

```js
function getAllFocusablesWithShadowRoots(shadowRoots) {
  const focusables = [];
  const root = new Focusable(document.documentElement, shadowRoots);
  let curr = root.nextFocusable();
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

### Returning a shadow host repeatedly for each internal focusable item

We considered having `nextFocusable()` step through every focusable item inside
an unexposed shadow tree while redacting each one to `{ shadowHost: host, target: null }`.
We rejected this approach for two reasons:
1. Calling `nextFocusable()` in a loop would return `{ shadowHost: host, target: null }`
   multiple times in a row (once per internal focusable item), producing
   duplicate entries with no new information, other than leaking the number of
   focusables within the shadow host.
2. Slotted light DOM elements are interleaved with shadow tree elements in focus
   order; exposing slotted elements while redacting shadow elements would cause
   `nextFocusable()` to alternate between light DOM targets and `{ shadowHost: host, target: null }`.

Treating the shadow tree (and its slotted light DOM) as a single entry by
default avoids both issues.

