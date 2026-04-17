# Overscroll Gestures Dev Trial Instructions.

## Summary

Last update: 2026-04-17

This document briefly outlines how to experiment with the **Overscroll
Gestures** feature. It also outlines known issues and requests for feedback.

Overscroll Gestures allows the developer to create areas of content that reside
in the overscroll region of another element. This is useful for effects like
side-menu, bottom-sheet, and pull to refresh. See the explainer linked below for
more detailed information.

## Trying it out

The feature is available in Chrome version **149.0.7797.0** and above -- latest
Chrome Canary will work. 

To enable the feature, enable Experimental Web Platform Features:
chrome://flags/#enable-experimental-web-platform-features

See the explainer and demos for how to use the feature from HTML/CSS/JS.

## Known issues

The feature is in early prototype stage. Some things are not implemented yet.
The following list of _unimplemented features_ is not exhaustive:

- toggle-modal command
- overscroll area inertness
- light dismiss
- ::backdrop suport
- automatic dialog / button activation

## Desired feedback

We'd love to hear what parts of this API work for you and which ones do not.
What is missing? What do you wish you could do with this and why can you not do
that?

For bugs that appear to be implementation bugs, please file a report at
https://crbug.com/new with `Overscroll Gestures` in the title of the issue.

For behaviors that seem to be choices that you disagree with, please file an
issue at https://github.com/openui/open-ui/issues

## Links

Chrome status: https://chromestatus.com/5261280285949952

Explainer: https://open-ui.org/components/overscroll-actions.explainer/

Demos: https://flackr.github.io/web-demos/html/overscrollcontainer/
