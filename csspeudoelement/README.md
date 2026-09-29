# `CSSPseudoElement.selectorText` Explainer

## Problem

The [`CSSPseudoElement`](https://drafts.csswg.org/css-pseudo-4/#csspseudoelement)
interface represents a pseudo-element in JavaScript, obtained via
`Element.pseudo(type)` (or `CSSPseudoElement.pseudo(type)` for nested
sub-pseudo-elements) and surfaced on events via `event.pseudoTarget`.

Historically, `CSSPseudoElement` only exposed a `type` attribute to identify the
pseudo-element, and was limited to simple, non-parameterized pseudo-elements
(`::before`, `::after`, and `::marker`). For those pseudo-elements, the base
type name (e.g., `"::before"`) is identical to the selector used to match it.

However, the web platform now includes—and is expanding `CSSPseudoElement` to
support ([#13804](https://github.com/w3c/csswg-drafts/issues/13804))—standardized
tree-abiding **functional (parameterized) pseudo-elements**, such as:

* **CSS Carousels**: `::scroll-button(inline-start)`,
  `::scroll-button(inline-end)`, `::scroll-button(left)`, etc.
* **View Transitions**: `::view-transition-group(name)`,
  `::view-transition-image-pair(name)`, `::view-transition-old(name)`, and
  `::view-transition-new(name)`.

As discussed in [CSSWG Issue #12161](https://github.com/w3c/csswg-drafts/issues/12161),
`CSSPseudoElement.type` represents only the **base type** of the pseudo-element
(e.g., `"::scroll-button"` or `"::view-transition-old"`), analogous to how an
element's tag name is distinct from its attributes or ID. Relying solely on
`type` creates two problems for parameterized pseudo-elements:

1. **Loss of identity / disambiguation**: Given a `CSSPseudoElement` (for
   example, from `event.pseudoTarget`), scripts cannot tell *which* scroll
   button (`inline-start` vs. `inline-end`) or *which* view transition snapshot
   (`header` vs. `avatar`) it represents.
2. **Broken round-tripping with string-based APIs**: Platform APIs that accept a
   `<pseudo-element-selector>` string—such as `element.pseudo()`,
   `getComputedStyle(element, pseudoElt)`, and
   `element.animate(keyframes, { pseudoElement })`—require the full selector
   including arguments. Passing `pseudo.type` (`"::view-transition-old"`) to
   these APIs fails to parse as a valid selector.

## Proposal

Following the CSSWG resolution in
[Issue #12161](https://github.com/w3c/csswg-drafts/issues/12161#issuecomment-3531224633)
and the specification text landed in
[PR #13169](https://github.com/w3c/csswg-drafts/pull/13169)
([CSS Pseudo-Elements Module Level 4 § 7.1](https://drafts.csswg.org/css-pseudo-4/#dom-csspseudoelement-selectortext)),
we propose implementing the `selectorText` attribute on `CSSPseudoElement`:

```idl
[Exposed=Window]
interface CSSPseudoElement {
  readonly attribute CSSOMString type;
  readonly attribute Element element;
  readonly attribute (Element or CSSPseudoElement) parent;
  readonly attribute CSSOMString selectorText;
  CSSPseudoElement? pseudo(CSSOMString type);
};
```

`CSSPseudoElement.selectorText` returns a `CSSOMString` representing the full,
normalized selector text used to select the pseudo-element, including the
pseudo-element name and any functional arguments, serialized in a form that can
round-trip. The attribute name mirrors `CSSStyleRule.selectorText` in CSSOM.

### How `selectorText` Differs from `type`

`type` and `selectorText` serve complementary roles:

* **`type`** identifies the *category/kind* of pseudo-element (the bare
  pseudo-element name without arguments). This makes it easy to check what class
  of pseudo-element an object belongs to without string parsing or prefix
  matching.
* **`selectorText`** identifies the *exact selector* for the pseudo-element
  instance (the base name plus normalized functional arguments). This uniquely
  identifies parameterized pseudo-elements on a given originating parent and can
  be passed directly into any API expecting a `<pseudo-element-selector>`.

| Input Selector | `pseudo.type` | `pseudo.selectorText` |
| :--- | :--- | :--- |
| `"::before"` | `"::before"` | `"::before"` |
| `":before"` (legacy syntax) | `"::before"` | `"::before"` |
| `"::backdrop"` | `"::backdrop"` | `"::backdrop"` |
| `"::scroll-marker"` | `"::scroll-marker"` | `"::scroll-marker"` |
| `"::scroll-button(left)"` | `"::scroll-button"` | `"::scroll-button(left)"` |
| `"::scroll-button( inline-end )"` | `"::scroll-button"` | `"::scroll-button(inline-end)"` |
| `"::view-transition-group(header)"` | `"::view-transition-group"` | `"::view-transition-group(header)"` |
| `"::view-transition-old(header)"` | `"::view-transition-old"` | `"::view-transition-old(header)"` |

For non-functional pseudo-elements (`::before`, `::after`, `::marker`,
`::backdrop`, `::scroll-marker`, `::view-transition`), `selectorText` and `type`
return the exact same string. For functional pseudo-elements, `type` omits the
parenthesized arguments while `selectorText` includes them.

### Normalization and Round-Tripping

Per spec, `selectorText` returns a normalized serialization that can round-trip:

* **Legacy single-colon selectors**: If a pseudo-element was requested using
  CSS2 single-colon syntax (e.g., `element.pseudo(":before")` or `":after"`),
  both `type` and `selectorText` normalize to modern double-colon syntax
  (`"::before"`, `"::after"`).
* **Whitespace and identifier escaping**: Functional arguments are serialized
  using standard CSSOM selector serialization rules (e.g., stripping extraneous
  whitespace such as `"::scroll-button( left )"` -> `"::scroll-button(left)"`).
* **Identity round-tripping**: For any `CSSPseudoElement` `pseudo`, passing
  `pseudo.selectorText` back to `.pseudo()` on its immediate originating parent
  returns the same `CSSPseudoElement` instance:
  ```js
  assert_equals(pseudo.parent.pseudo(pseudo.selectorText), pseudo);
  ```
* **Logical vs. physical `::scroll-button()` parameters**: Because
  `::scroll-button()` accepts both logical (`inline-start`, `inline-end`,
  `block-start`, `block-end`, `prev`, `next`) and physical (`up`, `down`,
  `left`, `right`) arguments that alias the same underlying pseudo-element
  depending on writing mode,
  [CSSWG Issue #14528](https://github.com/w3c/csswg-drafts/issues/14528)
  proposes normalizing the parameter to logical directions using the writing
  mode of the button (falling back to the default writing mode if the underlying
  pseudo-element does not exist yet or the originating element is disconnected).

## Examples

### Disambiguating `event.pseudoTarget` for parameterized pseudo-elements

When listening to events on an originating element, `type` can be used to match
all pseudo-elements of a given kind, while `selectorText` distinguishes specific
parameterized instances:

```js
carousel.addEventListener("click", (event) => {
  const pseudo = event.pseudoTarget;
  if (!pseudo) return;

  // Check the general kind of pseudo-element via `type`:
  if (pseudo.type === "::scroll-button") {
    console.log("A scroll button was clicked!");
  }

  // Check the specific scroll button via `selectorText`:
  if (pseudo.selectorText === "::scroll-button(inline-end)") {
    if (isAtEnd(carousel)) {
      loopToStart(carousel);
    }
  }
});
```

### Inspecting View Transition sub-pseudo-elements

In a view transition tree, multiple `::view-transition-group()`,
`::view-transition-image-pair()`, `::view-transition-old()`, and
`::view-transition-new()` pseudo-elements coexist under `::view-transition`.
`selectorText` preserves the `<pt-name-selector>` argument while `type`
identifies the role in the subtree:

```js
const vtRoot = document.documentElement.pseudo("::view-transition");
const headerGroup = vtRoot.pseudo("::view-transition-group(header)");
const headerPair = headerGroup.pseudo("::view-transition-image-pair(header)");
const headerOld = headerPair.pseudo("::view-transition-old(header)");

console.log(headerOld.type);         // "::view-transition-old"
console.log(headerOld.selectorText); // "::view-transition-old(header)"

// Round-tripping via `parent.pseudo(selectorText)`:
console.assert(headerOld.parent === headerPair);
console.assert(headerOld.parent.pseudo(headerOld.selectorText) === headerOld);
```

### Bridging `CSSPseudoElement` with string-based Web APIs

Several existing Web APIs take an originating `Element` and a pseudo-element
selector string rather than a `CSSPseudoElement` instance. `selectorText` allows
any `CSSPseudoElement` (including parameterized ones) to be passed into these
APIs without manual string reconstruction:

```js
function fadeOutPseudo(pseudo) {
  // Works for both "::before" and "::scroll-button(inline-end)"
  const currentOpacity =
      getComputedStyle(pseudo.element, pseudo.selectorText).opacity;

  return pseudo.element.animate(
      [{ opacity: currentOpacity }, { opacity: 0 }],
      {
        duration: 200,
        pseudoElement: pseudo.selectorText,
      });
}
```

## Alternatives Considered

During the discussion in
[CSSWG Issue #12161](https://github.com/w3c/csswg-drafts/issues/12161), two main
alternatives were considered:

### 1. Including arguments directly in `CSSPseudoElement.type`

We considered having `pseudo.type` return the full selector including arguments
(e.g., `pseudo.type === "::scroll-button(left)"`) instead of introducing a
separate property.

We decided against this because:
* It conflates the *type* of a pseudo-element with its *arguments*. If `type`
  included parameters, checking whether a pseudo-element is any scroll button or
  any `::view-transition-old` would require prefix checks
  (`pseudo.type.startsWith("::scroll-button(")`) or regex parsing.
* Keeping `type` as the bare pseudo-element name is conceptually consistent with
  DOM elements, where `Element.tagName` (or a type selector) is separate from
  element attributes, classes, or IDs.

### 2. Adding a dedicated `argument` or `arguments` property instead

We also considered exposing the functional parameters via a dedicated attribute—
either a nullable string (`readonly attribute CSSOMString? argument`) or a list
of strings (`readonly attribute sequence<CSSOMString>? arguments`)— returning
`"left"` or `["left"]` for `::scroll-button(left)` and `null` for `::after`.

While a structured arguments attribute may still be added in the future if
needed, the CSSWG resolved on `selectorText` first because:
* **Future syntax ambiguity**: Current parameterized pseudo-elements take a
  single argument, but future pseudo-elements may accept multiple arguments
  separated by spaces, commas, or more complex microsyntaxes. Committing to a
  single string vs. a sequence of strings now either creates legacy baggage or
  requires premature assumptions about how future pseudo-element arguments will
  be tokenized.
* **Ergonomics and round-tripping**: Even with `.type` and `.arguments`, authors
  would have to manually re-concatenate `type` and `arguments` to pass a
  selector string into `element.pseudo()`, `getComputedStyle()`, or
  `element.animate()`. `selectorText` directly provides the canonical,
  round-trippable selector string and aligns with existing CSSOM precedent
  (`CSSStyleRule.selectorText`).

## References

* [CSS Pseudo-Elements Module Level 4 — `CSSPseudoElement` interface](https://drafts.csswg.org/css-pseudo-4/#csspseudoelement)
* [CSSWG Issue #12161: Add a property to the `CSSPseudoElement` IDL interface to retrieve pseudo argument(s)](https://github.com/w3c/csswg-drafts/issues/12161)
* [CSSWG PR #13169: Define `.selectorText` for `CSSPseudoElement` IDL](https://github.com/w3c/csswg-drafts/pull/13169)
* [CSSWG Issue #13804: Add `::backdrop` and `::view-transitions` to `CSSPseudoElement`'s allowed pseudo-elements list](https://github.com/w3c/csswg-drafts/issues/13804)
* [CSSWG Issue #14528: `CSSPseudoElement.selectorText` parameter for `::scroll-button`](https://github.com/w3c/csswg-drafts/issues/14528)
