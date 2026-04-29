## overscroll-behavior: chain

### Problem

Currently there are 3 overscroll-behavior values:

* auto - which allows chaining, local and non-local boundary default actions
* contain - which allows local boundary default actions, but disallows chaining and non-local boundary default actions
* none - which disallows chaining, local and non-local boundary default actions

It would be nice to also have a value that

allows chaining and non-local boundary default actions disallows local boundary
default actions.

This would be a nice effect (or lack thereof) when bringing a side menu that
shouldn't overscroll and reveal the background, but still allow scroll chaining:
e.g. https://overscroll-menu-vmpstr.yoyo.codes/

### Proposal

We propose to add a new value to `overscroll-behavior`, namely `chain`. This
value would disable local border effects like bounce, but allow chaining.

### Alternatives 

One of the main alternatives is to simply do nothing or have a good browser
default. However, due to the small cost of a new value to an existing property,
we deemed it acceptable to add a new value.

### Initial proposal and resolution

https://github.com/w3c/csswg-drafts/issues/13370
