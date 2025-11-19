# Explainer: Declarative Overscroll Actions

## Introduction

The web platform allows for sophisticated scrolling experiences, but it
currently lacks a semantic way to utilize "overscroll" space (the area beyond
the scroll boundary).

Common UI patterns like **side-menus** (swiping past the edge to reveal a menu)
or **swipe-to-action** (swiping a list item to reveal delete buttons) currently
rely on complex nested scrollers or JavaScript gesture polyfills. These
workarounds are difficult to implement, computationally expensive, and often
fail to provide accessible alternatives for non-touch users.

## Proposal

We propose a set of HTML attributes that declaratively bind an element (the
"overscroll content") to the scroll boundary of a container.

Crucially, this binding is defined on an **activatable element** (like a
`<button>`). This ensures that every gesture-based action has a guaranteed,
accessible fallback interaction (click/Enter) without extra developer effort.

### Goals

* **Zero-JS Gestures:** Enable swipe-to-reveal and pull-to-refresh interactions
  using only HTML and CSS.
* **Accessibility by Default:** Enforce the existence of a semantic button to
  toggle the view, ensuring keyboard and assistive technology support.
* **Performance:** Offload gesture physics and animation to the browser's
  compositor thread (via scroll timelines).

## The API

We introduce attributes to bind a trigger button to both the container (the
scroll port) and the content (the element hidden in the overscroll area).

```html
<div id="container">
    <button overscroll-target="#menu" overscroll-host="#container">
      Open Menu
    </button>

    <menu id="menu">
        <li>Home</li>
        <li>Settings</li>
    </menu>
</div>
```

### Behavior

1.  **Positioning:** `#menu` is effectively absolutely positioned within the
    "overscroll" area of `#container`.
2.  **Chaining:** When a user scrolls `#container` to its limit, the scroll
    chains to the `#menu`, pulling it onto the screen.
3.  **Activation:** Clicking the `<button>` performs a `scrollIntoView`-like
    action on the `#menu`.

### Overlay Mode

By default, overscroll pushes the content. We also support an overlay mode
(similar to `position: sticky`) where the overscroll content slides *over* the
container.

```html
<button overscroll-target="#menu" overscroll-host="#container" overscroll-mode="overlay">
  Open Menu
</button>
```

## Events

To allow developers to hook into the lifecycle of the gesture (e.g., for refresh
logic or haptics), we expose the following events on the host container:

| Event Name | Description |
| :--- | :--- |
| `overscrollgesturestart` | Fired when the scroll boundary is breached and chaining begins. |
| `overscrollgesturechanging` | Fired when the gesture sufficiently drags overscroll to snap it to an open area (similar to `scrollsnapchanging`). |
| `overscrollgestureend` | Fired when the gesture completes and the state has changed. |
| `overscrollgesturecancel` | Fired when the gesture ends but snaps back to the original state. |

## Implementation Model

*Note: This section details the conceptual rendering tree structure.*

When configured, the browser constructs an internal box structure to handle
hit-testing and painting order:

_TODO: Update gox structure diagram._

![Box Structure Diagram](resources/box_structure.png)

1.  **`.container`** creates an internal `::overscroll-area-parent`.
2.  **`::overscroll-area-parent`** contains the overscroll elements (visually on
    top or bottom depending on mode).
3.  **`::overscroll-client-area`** holds the standard flow content.

**Hit Testing & Chaining:**

Hit testing begins at the `::overscroll-area-parent`. If no target is found in
the overscroll content, it recurses to the client area. Conversely, scroll
chaining bubbles up from the client area; when the client area limit is reached,
the delta is consumed by the `::overscroll-area-parent` to reveal the hidden
content.

![Overscroll Animation](resources/overscroll.gif)

([Simple polyfill on Codepen](https://flackr.github.io/web-demos/css-scroll-snap/menu2/))

## Accessibility Considerations

This proposal solves a major accessibility hurdle in gesture UIs. By attaching
the behavior to a `<button>`:

* **Keyboard Users:** Can tab to the button and activate it to reveal the
  menu/action.
* **Screen Readers:** Perceive a standard button connection rather than an
  invisible gesture zone.
* **Discoverability:** The button provides a visible affordance for the action.

## Alternatives Considered

### CSS-only Properties

We considered defining this relationship purely in CSS. While powerful, CSS
lacks the semantic enforcement of an interactive element. A CSS-only solution
runs the risk of creating "invisible" gestures that are inaccessible to users
who cannot perform swipe actions. By requiring an HTML activator, we enforce
progressive enhancement.

