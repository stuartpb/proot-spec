# HTML Parts and Walls

## Status

This document is "pre-specification" which means that I haven't even figured out enough spec bureaucracy to give this document a proper status. It's been [posted to the WICG Discourse](http://discourse.wicg.io/t/html-parts-and-roots/1032), and that's about it.

## Overview

This document proposes two new attributes that, like `class` and `id`, may be applied to any element in HTML:

- `wall`, a boolean attribute specifying that the element is to be treated as a boundary for the purposes of the `getPart()` function and the Walled Descendant Selector defined below.
- `part`, an attribute with a name value, following the same uniqueness and traversal constraints within walled boundaries as `id` has within document boundaries.
  - `id` may still be defined on an element with `part`, as they have different access implications. (A framework or polyfill may generate `id` values to uniquely identify every element in a document: this will not affect `part`.)

## Motivation

It's still not great to write components in HTML:

- CSS selectors always applying globally leads to unduly elaborate namespacing schemes like BEM.
- There's still no real good solution for addressing *unique children* of an element.

Shadow DOM is supposed to solve this, but it comes with a handful of even worse problems:

- The isolated context cuts you off from reusing your page styles.
- Just because I want to style for a component, doesn't mean I want to make its children inaccessible to `getElementsByTagName` &c.
- Styles defined by the page to apply *within* the context of Shadow DOM are impossible (the closest this came to a solution was `/deep/`, which only served to bring all the contextual problems of CSS selectors back and undo the most significant guarantee Shadow DOM is supposed to provide).
- Indexing elements *within* the component requires you to either go full-hog about designing the inner HTML to depend on Shadow DOM's ID isolation, or use clumsy class-based addressing techniques.
- Shadow DOM contexts can't be created declaratively (without Custom Elements, which require lots of JS and bring further constraints).

Walled elements introduce a lighter option than Shadow DOM contexts for *basic* encapsulation. (For styling within heavier encapsulation, the Walled Descendant selector may *also* be used for *crossing* Shadow DOM boundaries.) It provides a facility page authors may use to create a hierarchy *in the context of their page*, allowing them to have simple and straightforward access *within the boundaries of their hierarchy*, without being concerned about namespacing *at every level* (the scope of concern for namespace schemes can be limited to selecting the proot).

## DOM Parts

Elements have a `getPart()` function, which works similarly to how `getElementById` or `getElementByClassName[0]` works (doing a depth-first search under the element), except looking for `part` rather than `id` and avoiding traversal beyond wall boundaries (as well as the document boundaries which all traversal algorithms avoid).

Example: say I have this HTML:

```html
<div id="example-lotto" class="lotto" wall>
  <div class="lid">
    <span part="timer" id="example-lotto-timer">108:00</span>
  </div>
  <div part="bubble" wall id="example-lotto-bubble">
    <div part="funnel" id="example-bubble-funnel">
      <span class="ball">42</span>
    </div>
    <div part="track" id="example-running-track">
      <span class="ball">23</span>
      <span class="ball">16</span>
    </div>
  </div>
  <div part="track" id="example-chosen-track">
    <span class="ball">4</span>
    <span class="ball">8</span>
    <span class="ball">15</span>
  </div>
</div>
```

(Note: This element would normally be created as one of many, having no IDs defined. The IDs are purely for illustrative purposes in the following description.)

Let `exampleLotto = document.getElementById('example-lotto')`.

Running `exampleLotto.getPart('timer')` would return the element with the ID `example-lotto-timer`, as that is the unique (first) element under `exampleLotto` with the `part` attribute value `timer`.

Running `exampleLotto.getPart('bubble')` would return the element with the ID `example-lotto-bubble`, as that is the unique (first) element under `exampleLotto` with the `part` attribute value `bubble`. (The fact that the element itself has a `wall` value does not stop the *element itself* from being found.)

Running `exampleLotto.getPart('funnel')` would return `null`, as the only element with the `part` attribute value `funnel` underneath `exampleLotto` is underneath the element with the ID `example-lotto-bubble`, which has the `wall` attribute defined, causing the search to not recurse further into that element's descendants. (Getting the element with the ID `example-bubble-funnel` would require starting the search from its first ancestor with the `wall` attribute, namely the element with the ID `example-lotto-funnel`, ie. `exampleLotto.getPart('bubble').getPart('funnel')`.

Running `exampleLotto.getPart('track')` would return would return the element with the ID `example-chosen-track`, as that is the unique (first) element under `exampleLotto` with the `part` attribute value `track` that is not underneath an element (namely, the element with the ID `example-lotto-bubble`) which has the `wall` attribute defined (which causes the search to not recurse further into that element's descendants, hence skipping the element with the ID `example-running-track`).

Running `exampleLotto.getPart('bubble').getPart('track')` would return the element with the ID `example-running-track`, as that is the unique (first) element under `exampleLotto.getPart('bubble')` with the `part` attribute value `track`.

### Alternate part access paradigm

Alternately, elements could have a `parts` object with an accessor that follows the logic described for `getPart()` above, providing each `part` inside the wall as a property, similar to how `dataset` provides `data-` attributes or `window` (unreliably) provides elements by ID. (I honestly think UAs should provide both - I feel there are enough specs forcing authors to use their opinion-of-the-week access pattern that doesn't match the rest of what the author is working with.)

## Part Boundary Selector

A `|>` sequence may be used after a selector specifying a element, ID, and/or class, to define a wall or Shadow DOM boundary crossing. Selectors to the right of the `|>` may only specify elements, classes, or part or ID names: the `#` character, after a `|>`, refers to either IDs or parts within that DOM / wall. It *must not cross* further walls.

Note: It is important to note that the part *left of the walled desecendant selector* may - like all other selectors - traverse *any number* of walls. Walls *only* come into play *around the walled descendant selector*. Any non-walled descendant selectors (namely, the whitespace "general descendant" selector), as before, *are not separated* by walls.

## Differences between walled elements and Shadow DOM

- Elements under a walled element are still selected by selectors in the top-level document, as well as functions like `getElementsByClassName`.
- Walls can be defined without JS.
- Child elements may be moved between walled elements without having to create a copy with `importNode`, discard the old node, and expecting things that `importNode` doesn't copy to break.

## Differences between `part` and `class` or `name`

- Unlike class names, parts are expected to be unique within the context of their wall (which allows for optimizations to `getPart`, as well as warnings in validators, and other developer tool enhancements).
- Part names under walls do *not* have to be considered in the global namespace, unlike class names. (Similarly, part names do *not* have to correspond to the name used for a value when submitting a form.)
- Unlike `getElementsByName` or `getElementsByClassName`, `getPart` is restricted to wall boundaries.

## Extensions to existing elements

A standard structure for the standard elements UAs currently construct within Shadow DOM could be defined, using `part` attributes to refer to a hierarchy of their components, for access in styling, as an alternative to non-standard vendor-prefixed pseudo-elements, as we currently use. (This includes the sub-components of scrollbars, which would remain pseudo-elements, but be standardized as pseudo-elements containing walls.)

## Example: Doctor Listings 

```html
<html>
<head>
<style>
/* TODO: finish example stylesheet */
.profile/#photo {

}
.profile/#name {

}
.profile/#bio {

}
</style>
</head>
<body>
<div id="profiles">
  <div class="profile" root>
    <img src="person-a.jpg" part="avatar">
    <div part="name">Foo Bar</div>
    <p part="bio">lorem ipsum</p>
  </div>
  <div class="profile" root>
    <img src="person-b.jpg" part="avatar">
    <div part="name">Boo Car</div>
    <p part="bio">lorem ipsum</p>
  </div>
  <div class="profile" root>
    <img src="person-c.jpg" part="avatar">
    <div part="name">Kenny Baz</div>
    <p part="bio">lorem ipsum</p>
  </div>
</div>
</body>
</html>
```

## Example: Chat UI (initial content with HTML template)

```html
<html>
<head>
<style>
/* TODO: finish example stylesheet */
.message-card/#avatar {

}
.message-card/#name {

}
.message-card/#message {

}
</style>
<template id="message-card-template">
  <div class="message-card" root>
    <img part="avatar">
    <div part="name"></div>
    <div part="message"></div>
  </div>
</template>
</head>
<body>
<div id="messages"></div>
<script>
// TODO: write script
</script>
</body>
</html>
```

## Example: Chat UI (live snapshot of variant using no HTML template)

```html
<html>
<head>
<link rel="stylesheet" href="same-style-as-last-template.css">
</head>
<body>
<div id="messages">
  <div class="message-card" id="7cf00c89-800f-495b-9e44-a2ffd9df817c" root>
  </div>
  <div class="message-card" id="26888874-dbe9-4efc-8f88-873af7139a4d" root>
  </div>
  <div class="message-card" id="5c212dc7-5d73-4759-a466-589a08d5a98e" root>
  </div>
  <div class="message-card" id="ed8818ff-353d-4219-a28f-a97b531e95f2" root>
  </div>
  <div class="message-card" id="331bb13b-7071-4d69-9b04-fb62502a5601" root>
  </div>
  <div class="message-card" id="7fc6ee0c-85d2-47e2-a8f4-4f499d44bfdb" root>
  </div>
  <div class="message-card" id="95ca156f-8e59-4372-8e6f-bff9c8a7c4ac" root>
  </div>
  <div class="message-card" id="cd3ac0ae-925f-45e8-a1e7-98e1b215ba9e" root>
  </div>
</div>
<script>
// TODO: write script
</script>
</body>
</html>
```

## TODO

- Everything marked "TODO" above.
- Add proposals for what the standard element parts look like.
