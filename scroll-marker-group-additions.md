# Element-Backed Scroll Marker Groups and Scroll Markers

## Authors

- Vladimir Levin
- Keith Cirkel
- Rob Flack

## Motivation

In [CSS Overflow 5](https://drafts.csswg.org/css-overflow-5/#scroll-markers), a
scroll container can opt into generating a scroll marker group via the
`scroll-marker-group` property:

```css
scroll-marker-group: none | [ [ before | after ] || [ links | tabs ] ];
```

Setting `before` or `after` creates a `::scroll-marker-group` pseudo-element box
as an immediate sibling of the scroll container (or as the first/last child when
originating on the root element), operating in either `links` mode (the default)
or `tabs` mode. Descendants of the scroller (or `::column` fragments in a
multi-column scroller) whose `::scroll-marker` pseudo-element has a computed
`content` other than `none` then generate `::scroll-marker` boxes inside that
group.

While pure pseudo-elements work well for simple dot indicators and
self-contained carousels, authors run into two structural limitations in more
complex interfaces.

First, `before` and `after` can only place the `::scroll-marker-group` box
immediately adjacent to the scroll container. In practice, designs frequently
place pagination dots or tab bars inside a separate `<header>`, within a shared
toolbar alongside headings and action buttons, or in an unrelated grid track.
Positioning a sibling pseudo-element into another container's layout requires
fragile CSS Anchor Positioning or fixed-size reservations.

Second, `::scroll-marker` pseudo-elements cannot contain rich HTML subtrees
(such as `<picture>` thumbnails, inline `<svg>` icons, structured multi-line
labels, or framework-managed markup) and cannot have event listeners attached
directly to individual markers. Although `scroll-target-group: auto` lets
authors group ordinary `<a href="#id">` links, it requires maintaining unique
IDs for every target, only applies to anchor elements rather than `<button>`
controls, does not provide `tabs` mode semantics (`tablist`/`tab`/`tabpanel`
roles, roving `focusgroup` navigation, and active-panel focus scoping), and
cannot adapt when markers are generated dynamically from layout such as
`::column::scroll-marker`.

## Proposal Overview

We propose a two-part extension that lets authors progressively back
`::scroll-marker-group` and `::scroll-marker` with real DOM elements:

1. **Element-backed marker groups:** Extend `scroll-marker-group` to accept an
   `<id-selector>` (alongside `links` or `tabs`) referencing an element in the
   page marked with a `scrollmarkergroup` attribute. That element becomes the
   scroll marker group instead of creating a new pseudo-element box, while
   `::scroll-marker-group` continues to work as a selector matching that
   element.
2. **Element-backed markers:** Direct children of that `scrollmarkergroup`
   element can be marked with a `scrollmarker` attribute. When scroll markers
   are generated for the scroller, they bind in DOM order to those
   `scrollmarker` elements so `::scroll-marker` refers to the first, second,
   third, etc. real element instead of creating its own box. If there are fewer
   `scrollmarker` elements than markers needed, the remaining markers are
   generated as `::scroll-marker` pseudo-elements appended as the last children
   of the `scrollmarkergroup` element. If there are more `scrollmarker` elements
   than needed, the extra ones become `display: none`.

## Referencing a Scroll Marker Group Element

The `scroll-marker-group` property grammar is extended with `<id-selector>`:

```
Name: scroll-marker-group
Value: none | [ [ before | after | <id-selector> ] || [ links | tabs ] ]
```

As with `before` and `after`, omitting the mode keyword defaults to `links`.
When an `<id-selector>` is specified, the user agent looks up the first element
in tree order within the same tree scope matching that ID. If that element has
the boolean `scrollmarkergroup` HTML attribute, it becomes the scroll
container's scroll marker group container.

```html
<header class="gallery-header">
  <h2>Featured Work</h2>
  <div id="gallery-markers" scrollmarkergroup></div>
</header>

<div class="gallery">
  <figure class="slide"><img src="1.jpg" alt="Project 1"></figure>
  <figure class="slide"><img src="2.jpg" alt="Project 2"></figure>
  <figure class="slide"><img src="3.jpg" alt="Project 3"></figure>
</div>
```

```css
.gallery {
  overflow-x: auto;
  scroll-snap-type: x mandatory;
  scroll-marker-group: #gallery-markers tabs;
}

.gallery::scroll-marker-group {
  display: flex;
  gap: 0.5rem;
}

.slide::scroll-marker {
  content: "";
  width: 0.75rem;
  height: 0.75rem;
  border-radius: 50%;
}

.slide::scroll-marker:target-current {
  background: currentColor;
}
```

In this example, `.gallery` does not create a sibling pseudo-element box, and
`#gallery-markers` stays where it is inside `.gallery-header`. Instead,
`#gallery-markers` serves as the scroll marker group container, acquiring the
implicit `tablist` role and roving focusgroup behavior from `tabs` mode, and
the three generated `.slide::scroll-marker` pseudo-element boxes are created as
children of `#gallery-markers`. Because `#gallery-markers` is a real DOM
element, those `tabs` and `links` semantics act as implicit defaults: explicit
`role`, `aria-*`, `tabindex`, `focusgroup`, and other HTML attributes on the
element take priority, allowing authors to override or customize the `tabs` or
`links` behavior as appropriate.

The `.gallery::scroll-marker-group` selector still matches, resolving directly
to `#gallery-markers` so both `.gallery::scroll-marker-group` and regular
selectors like `#gallery-markers` apply to the same element, while style
inheritance continues to flow from `#gallery-markers`'s actual parent in the
flat tree (`.gallery-header`) rather than from `.gallery`. How pseudo-element
and regular element rules are prioritized in the cascade, and whether UA size
containment applies, are discussed in [Open Questions](#open-questions).

### Unmatched References, Conflicts, and Unbound Groups

Requiring the explicit `scrollmarkergroup` attribute ensures that a stylesheet
cannot accidentally turn an arbitrary page element into an interactive
navigation or tablist widget. If the `<id-selector>` does not match any element
in the same tree scope, or if the matched element does not have the
`scrollmarkergroup` attribute, no scroll marker group is established and the
scroller behaves as if `scroll-marker-group: none` were set:

```html
<!-- Missing `scrollmarkergroup`: #not-a-group is ignored and left untouched -->
<div id="not-a-group">Static content</div>
<div class="scroller" style="scroll-marker-group: #not-a-group;">...</div>
```

If multiple scroll containers reference the same `scrollmarkergroup` element,
the first scroll container in tree order claims it; subsequent scroll containers
treat their `scroll-marker-group` as `none`. Conversely, if an element carries
`scrollmarkergroup` in markup while no scroll container references it (for
instance, when a `@media` query disables carousel layout on print or wide
screens), it simply renders as a normal HTML element.

## Backing Scroll Markers with Child Elements

Once a scroll container is bound to a `scrollmarkergroup` element, direct
children of that element marked with the boolean `scrollmarker` attribute can
stand in for generated `::scroll-marker` boxes. Only direct children with
`scrollmarker` participate; nested elements deeper in the subtree are ignored.

Items inside the scroll container (or `::column` fragments) still opt into
having a scroll marker the same way they do today: by computing `content` on
`::scroll-marker` to a value other than `none` (commonly `content: ""`). As the
scroller collects these markers in tree order, it pairs them sequentially with
the direct `[scrollmarker]` children of the `scrollmarkergroup` element in DOM
order. When a marker pairs with a real `[scrollmarker]` element, its
`::scroll-marker` pseudo-element refers to that element instead of creating a
new box, and the element preserves its own DOM children. The `content` value on
`::scroll-marker` is only used when synthesizing fallback pseudo-element boxes.

### Matching Marker Counts

When the number of `[scrollmarker]` children matches the number of markers
produced by the scroller, each marker binds one-to-one with its corresponding
child element:

```html
<div id="product-tabs" scrollmarkergroup>
  <button scrollmarker>
    <img src="front-thumb.jpg" alt="">
    <span>Front</span>
  </button>
  <button scrollmarker>
    <img src="side-thumb.jpg" alt="">
    <span>Side</span>
  </button>
  <button scrollmarker>
    <img src="interior-thumb.jpg" alt="">
    <span>Interior</span>
  </button>
</div>

<div class="product-views">
  <section class="view"><img src="front.jpg" alt="Front view"></section>
  <section class="view"><img src="side.jpg" alt="Side view"></section>
  <section class="view"><img src="interior.jpg" alt="Interior view"></section>
</div>
```

```css
.product-views {
  overflow-x: auto;
  scroll-snap-type: x mandatory;
  scroll-marker-group: #product-tabs tabs;
}

.view::scroll-marker {
  content: "";
}

#product-tabs > [scrollmarker]:target-current {
  border-color: Highlight;
}

/* Equivalent: `.view::scroll-marker` selects the same <button> elements */
.view::scroll-marker:target-before {
  opacity: 0.7;
}
```

Here, the three `.view` sections pair with the three `<button scrollmarker>`
elements in order without needing IDs on each section. Each `<button>` keeps its
thumbnail `<img>` and `<span>` children, receives `tab` semantics and activation
behavior that scrolls its paired `.view` into view, and matches
`:target-current`, `:target-before`, and `:target-after` as the user scrolls.
Just as with `::scroll-marker-group`, rules targeting `.view::scroll-marker` and
rules targeting `#product-tabs > button` match the same underlying element,
which continues to inherit styles from `#product-tabs`.

### Fewer `scrollmarker` Elements Than Needed

If the scroll container needs more scroll markers than there are direct
`[scrollmarker]` children, the available `[scrollmarker]` elements are used
first in DOM order, and the remaining markers are generated as `::scroll-marker`
pseudo-element boxes appended as the last children of the `scrollmarkergroup`
element:

```html
<div id="deck-nav" scrollmarkergroup>
  <span class="nav-label">Jump to:</span>
  <button scrollmarker>Overview</button>
  <button scrollmarker>Architecture</button>
</div>

<div class="deck">
  <article class="slide">...</article>
  <article class="slide">...</article>
  <article class="slide">...</article>
  <article class="slide">...</article>
</div>
```

```css
.deck {
  overflow-x: auto;
  scroll-snap-type: x mandatory;
  scroll-marker-group: #deck-nav links;
}

.slide::scroll-marker {
  content: "•";
  padding-inline: 0.25rem;
}
```

Direct children without `scrollmarker`, such as `<span class="nav-label">`, are
left in place and skipped when pairing markers. The first two `.slide` elements
bind to the `"Overview"` and `"Architecture"` buttons, preserving their text.
Because there are no `[scrollmarker]` elements left for the third and fourth
slides, those two slides generate `::scroll-marker` pseudo-element boxes
(populated with `content: "•"`) as the last children of `#deck-nav`, after
`<button scrollmarker>Architecture</button>`.

### More `scrollmarker` Elements Than Needed

Conversely, if the `scrollmarkergroup` element has more direct `[scrollmarker]`
children than the scroller needs, the unassociated trailing `[scrollmarker]`
elements become `display: none`. This is particularly useful when marker counts
depend on layout, such as column pagination with `::column::scroll-marker`:

```html
<div id="page-markers" scrollmarkergroup>
  <button scrollmarker>Page 1</button>
  <button scrollmarker>Page 2</button>
  <button scrollmarker>Page 3</button>
  <button scrollmarker>Page 4</button>
</div>

<div class="article-pager">
  <p>Long-form content fragmented into viewport-sized columns...</p>
</div>
```

```css
.article-pager {
  overflow-x: auto;
  columns: 1;
  scroll-snap-type: x mandatory;
  scroll-marker-group: #page-markers tabs;
}

.article-pager::column {
  scroll-snap-align: start;
}

.article-pager::column::scroll-marker {
  content: "•";
}
```

Suppose on a wide screen `.article-pager` fragments into only two columns.
Columns 1 and 2 bind to the `Page 1` and `Page 2` buttons, while the `Page 3`
and `Page 4` buttons become `display: none`, generating no boxes and dropping
out of focus order and the accessibility tree. If the viewport narrows so the
content fragments into four columns, `Page 3` and `Page 4` automatically
reappear and bind to the new columns; if it narrows further into five columns,
the fifth column appends a fallback `::scroll-marker` box (`"•"`) after
`Page 4`. If `scroll-marker-group` is later set to `none`, `#page-markers` is no
longer bound to a scroller and all four buttons return to their normal display.

## Focus, Events, and Accessibility

Both `links` and `tabs` modes operate uniformly across element-backed markers
and any trailing generated pseudo-element markers within the same group. In
`links` mode, the `scrollmarkergroup` element receives an implicit `navigation`
role and every active marker acts as a sequential tab stop with `link`
semantics, scrolling its target into view on activation and moving the
sequential focus navigation starting point to that target. In `tabs` mode, the
`scrollmarkergroup` element receives an implicit `tablist` role and forms a
single roving `focusgroup` across both its bound `[scrollmarker]` children and
any generated fallback `::scroll-marker` pseudo-elements: only the
`:target-current` marker is in the `Tab` order, arrow keys cycle through all
markers in the group, target elements receive `tabpanel` semantics, and pressing
`Tab` from the active tab moves focus into its associated target panel.

Because `[scrollmarkergroup]` and `[scrollmarker]` are real DOM elements, those
mode-assigned roles and states serve only as implicit defaults. Explicit `role`
and `aria-*` attributes take priority, letting authors customize or enrich the
accessibility tree in ways that pure pseudo-elements cannot, such as naming the
group via `aria-labelledby` or labeling icon-only markers with `aria-label`:

```html
<h2 id="views-heading">Vehicle Views</h2>
<div id="view-markers" scrollmarkergroup aria-labelledby="views-heading">
  <button scrollmarker aria-label="Front exterior view">
    <svg aria-hidden="true"><!-- icon --></svg>
  </button>
  <button scrollmarker aria-label="Interior cabin view">
    <svg aria-hidden="true"><!-- icon --></svg>
  </button>
</div>
```

Unlike `scroll-marker-group: before` and `after`, which synthetically position
the generated group immediately before or after the scroll container in
sequential focus order, an `<id-selector>` group remains at its natural DOM
position in focus order (aside from `tabs` mode directing `Tab` from the active
tab into its target panel). Finally, because `[scrollmarkergroup]` and
`[scrollmarker]` nodes are real DOM elements, focusing or activating them
exposes the element itself as `document.activeElement` and `event.target`,
whereas any trailing generated `::scroll-marker` pseudo-elements keep standard
pseudo-element targeting (`document.activeElement` on the scroller,
`event.target` on the originating element, and `event.pseudoTarget` identifying
the pseudo-element).

## Why Not `<scrollmarkergroup>` and `<scrollmarker>` Elements?

An obvious alternative is introducing dedicated `<scrollmarkergroup>` and
`<scrollmarker>` HTML elements instead of attributes. We prefer attributes
because scroll marker groups and markers are behavioral roles that authors often
want to layer onto existing semantic HTML elements rather than replace them
with a single tag.

For the group container, `<nav>` is often the right element for section
navigation, `<ol>` or `<ul>` fits a numbered or unnumbered step list, and
`<menu>` or `<div>` suits a toolbar or tab strip. Marking
`<nav scrollmarkergroup>` directly avoids extra wrapper elements and keeps the
intended landmark or list semantics intact when carousel styles are inactive
(for example, outside a `@media` query or before stylesheets load).

The same applies to individual markers. Depending on the control and its
fallback behavior, a marker may naturally be a `<button>`, an `<a href="#...">`
link that still navigates without CSS, or an `<li>` inside an `<ol>` or `<ul>`
where HTML's content model and parser only allow `<li>` as direct children:

```html
<nav id="chapter-nav" aria-label="Chapters" scrollmarkergroup>
  <a href="#intro" scrollmarker>Introduction</a>
  <a href="#methods" scrollmarker>Methods</a>
  <a href="#results" scrollmarker>Results</a>
</nav>
```

When a scroll container binds to `#chapter-nav`, the `<nav>` and its links
participate as a full scroll marker group with `:target-current` tracking and
fallback marker generation. When unbound, the markup remains an ordinary,
functional `<nav>` of anchor links with no inert custom tags or redundant
wrappers.

## Open Questions

### Pseudo-Elements Aliasing Real Elements and Cascade Priority

Having `::scroll-marker-group` and `::scroll-marker` refer to existing DOM
elements introduces a new mechanism in CSS where a pseudo-element selector
aliases a real element rather than selecting a separate generated sub-element.
We need to define how declarations from pseudo-element selectors cascade
relative to declarations from regular selectors matching the same element:

```css
#product-tabs > button {
  background: white;
}

.view::scroll-marker {
  background: lightgray;
}
```

One option is to merge both sets of rules in the normal author cascade and
compare specificity and source order directly. Another is to give aliased
pseudo-element rules a distinct cascade priority (for instance, cascading below
direct element rules as defaults, or above them).

Aliasing also introduces a style-resolution ordering dependency: determining
which `.view::scroll-marker` (or `.pager::column::scroll-marker`) maps to the
*n*th `[scrollmarker]` child depends on the scroller's computed
`scroll-marker-group`, the computed `content` of descendants' `::scroll-marker`
rules, and, in the case of `::column`, layout itself.

### Layout Dependencies and Size Containment

For generated `before` and `after` groups, CSS Overflow 5 recommends
`::scroll-marker-group { contain: size !important; }` in the UA stylesheet so
creating or updating `::scroll-marker` boxes cannot invalidate layout outside
the group. With real `[scrollmarkergroup]` and `[scrollmarker]` elements,
changing the number of active markers can affect layout when unused markers
become `display: none` or fallback pseudo-elements are appended:

```html
<div style="display: flex; flex-direction: column; height: 100vh;">
  <div id="markers" scrollmarkergroup>
    <button scrollmarker>Page 1</button>
    <button scrollmarker>Page 2</button>
    <button scrollmarker>Page 3</button>
  </div>
  <div class="pager" style="flex: 1; overflow-x: auto; columns: 1;">...</div>
</div>
```

If `#markers` wraps onto two lines when all three buttons are shown, `.pager`
gets less block size, which can change how content fragments into columns. If
that new column count hides `Page 3` with `display: none`, `#markers` shrinks
back to one line, growing `.pager` again and risking a layout cycle. Similar
feedback loops can happen if `.pager::column::scroll-marker` applies
size-affecting styles to `[scrollmarker]` elements.

Applying `contain: size` (or `contain: size !important`) to
`[scrollmarkergroup]` elements avoids these cycles, at the cost of breaking
intrinsic sizing when authors expect a `<div scrollmarkergroup>` to size
naturally around its child buttons. Alternatives include requiring
`contain: size` only when `::column::scroll-marker` is involved, or latching
marker counts across layout passes.

### Overriding `display: none` on Unused Markers

When there are more `[scrollmarker]` children than markers needed, the excess
elements become `display: none`. It is an open question whether authors can or
should be able to override that `display: none`.

If unused markers are hidden via an ordinary UA stylesheet rule without
`!important`, any author rule setting `display` on markers would unintentionally
keep unused markers visible:

```css
/* Would this accidentally unhide unused [scrollmarker] buttons? */
#page-markers > [scrollmarker] {
  display: inline-flex;
}
```

Enforcing `display: none !important` in the UA stylesheet (or suppressing box
generation directly) avoids that footgun so author `display: inline-flex` rules
only affect active markers. On the other hand, some designs may want unused
`[scrollmarker]` elements to remain visible in a disabled or placeholder state
rather than disappearing. Supporting that intentionally would require deciding
both how authors opt into showing unused markers (for example, matching
`:disabled` or a dedicated pseudo-class instead of forcing `display: none`) and
what activation or focus behavior an unbound marker should have.
