# ViewTransition.waitUntil()

## Problem

The ViewTransition automatically constructs a pseudo-element tree to display and
animate participating elements in the transition. Per spec, this subtree is
constructed when the view transition starts animating and is destroyed when the
animations associated with all view transition pseudo-elements are in the
finished state (or more precisely in a non-running non-paused state). 

This works for a vast majority of cases and provides a seamless experience for
the developers. However, for more advanced cases, this is insufficient as there
are times when developers want the view transition pseudo-tree to persist beyond
the animation finish state.

One example is tying view transitions with Scroll Driven Animations. When the
animation is controlled by a scroll timeline, we don't want the subtree to be
destroyed when the animations finish since scrolling back should still be able
to animate the pseudo elements.

## Proposal

In order to enable advanced uses of view transition, we propose to add a
`waitUntil` function on the ViewTransition object which takes a promise. This
promise then delays destruction of the pseudo-tree until it is settled.

```idl
struct ViewTransition {
 ...

 // Delay the termination of the view transition until `p` settles.
 [CallWith=ScriptState] void waitUntil(Promise<any> p);

 ...
}
```

This allows arbitrary extension for the view transition, controlled by the
developer.

Note that the naming and behavior mimics [ServiceWorker's waitUntil
method](https://w3c.github.io/ServiceWorker/#wait-until-method)

## Example

```js
let finishTransition;
function startTransition() {
    let transition = document.startViewTransition();
    transition.waitUntil(new Promise(r => { finishTransition = r; });
}

// The transition will persist while arbitrary animations and work is done.
...

// To allow the transition to end, resolve the finish promise.
finishTransition();
```

## Alternatives considered

This is an advanced feature that can be used for several disjoint cases.

We have considered bespoke scroll driven animation solutions, such as monitoring
which timeline the animations are associated with. However, this doesn't allow
easy rAF-based animations.

Overall we decided to go with the `waitUntil()` approach, mimicking
ServiceWorker's prior art for a similar feature.
