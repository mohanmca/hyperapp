# Implementation Notes

## Application Architecture

### Setup

Declare all our variables. Set `appRoot` to document.body if no root is supplied.

Set `element` to the root's first child element. If you are hydrating, this is how we can target server side rendered DOM nodes instead of appending to the container as usual.

Iterate `appMixins` using `props` as the first mixin. Mixins we find we append to `appMixins` changing the length of the list and extending the life of the loop.

Inside the loop we also build the array of event handlers and initialize the state and actions.

### The Initial Render

Before the initial render, fire [load](/docs/events.md#load), pass it the element and save the result in `oldNode`.

This allows you to iterate the root child nodes inside load and return a [virtual node](/docs/virtual-node.md) we can use for [hydration](/docs/hydration.md).

If you don't return anything from an event, [`emit`](#emit) returns the same data we pass it. This way we can invalidate `oldNode` and `element`. Otherwise, [`patch`](#patch) will overwrite the first child node inside the root.

Note that calling actions inside load will trigger a state update, but it will not cause the view to be rendered since we still haven't called `requestRender` at this point.

Call [`requestRender`](#requestRender) so the view is rendered on the next repaint.

Finally, return `emit` to the caller. This allows you to communicate with the application from the outside via [custom events](/docs/events.md#custom-events).

### `initialize`

Iterate the supplied actions injecting our state update logic. If the current action is not a function, recurse into the namespace.

Before calling an action, fire [action](/docs/events.md#action) with the action information and ignore the result.

Inside the wrapped action, call the unwrapped action function with the global state, actions and data payload. Then fire [resolve](/docs/events.md#resolve) with the result allowing you to manipulate it before triggering a state update.

If the result of an action is a function, call it immediately with [`update`](#update), otherwise call `update` with the result.

### `render`

Call the view to produce a new virtual node and run [`patch`](#patch) with it and the old node. Save the new node in `oldNode` for the next patch.

Flush `globalInvokeLaterStack` and call each oncreate and onupdate event handler added during patching.

### `requestRender`

Request [`render`](#render) to be called on the next repaint using [requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame).

An application may receive a request to render while another render is running or may not even have a view. Skip over those cases.

If a render is correctly scheduled, toggle `willRender` so that future requests are disregarded until we're done with the current one.

### `update`

Skip over null and undefined states, e.g., when an action returns null or undefined. Then, use our shallow [`merge`](#merge) on the global state and new state or partial state and use the result to fire [update](/docs/events.md#update).

You can validate or modify the state inside this event to overwrite the global state. If you return `false`, we won't update the state or request a new render.

Return `appState`.

### `emit`

Call each subscribed handler of the specified event with the supplied data. The result of the event handler will be used to overwrite the data, except for null and undefined returns.

After we've called all the subscribers return the new data.

### `merge`

Shallow, non-destructive merge prioritizing on the second argument.

## Virtual DOM

The function of the Virtual DOM is to find the differences between `oldNode` and the new `node` and update the DOM with this information as efficiently as possible.

### `getKey`

Retrieve a key from the supplied node. Keys help `patch` identify which nodes refer to the same element between patches. Keys need not be unique across the entire tree, but only between siblings.

### `setData`

Set or remove an element's property and attribute. Some properties are read-only, so we wrap the property set in a try-block to ignore invalid cases.

### `createElement`

Create a new DOM element with [createTextNode](https://developer.mozilla.org/en-US/docs/Web/API/Document/createTextNode) for nodes that are a string or [createElement](https://developer.mozilla.org/en-US/docs/Web/API/Document/createElement) otherwise. If the element is an SVG element or we are currently patching an SVG element, we'll use [createElementNS](https://developer.mozilla.org/en-US/docs/Web/API/Document/createElementNS) instead.

Push any oncreate handlers to `globalInvokeLaterStack` to call them after patch is done.

### `updateElement`

Iterate the old and new data attributes and properties of the element we are currently patching and determine whether they have changed or not. If they have changed, call [`setData`](#setdata) to update the DOM.

Push any onupdate handlers to `globalInvokeLaterStack` to call them after patch is done.

### `removeElement`

If the element doesn't have an onremove handler, remove the element with [removeChild](https://developer.mozilla.org/en-US/docs/Web/API/Node/removeChild), otherwise call the function. You are responsible for removing the element inside it.

### `patch`

Patch the supplied element and its children recursively using the old and new virtual nodes.

If the old node is undefined, create a new element using [createElement](#createelement) and update the element.

If the old node exists and has the same tag name as the new node, update the element data and patch its children. This process consists of 4 steps:

- Create a map with the old keyed nodes.
- Update the element's children.
- Remove any remaining unkeyed old nodes.
- Remove any unused keyed old nodes.

If all the above are false, we know that both nodes are either text nodes or their tag names don't match.

Make sure the element is not undefined and replace it with a new element.

If the new node matches the element's value, that means we are comparing two text nodes that have the same value, so we can skip over.

Return the element.

