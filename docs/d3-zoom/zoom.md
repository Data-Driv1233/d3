# Zoom behavior

[Examples](https://observablehq.com/collection/@d3/d3-zoom) · The zoom behavior handles a variety of interaction events:

| Event        | Listening Element | Zoom Event  | Default Prevented? |
| ------------ | ----------------- | ----------- | ------------------ |
| mousedown⁵   | selection         | start       | no¹                |
| mousemove²   | window¹           | zoom        | yes                |
| mouseup²     | window¹           | end         | yes                |
| dragstart²   | window            | -           | yes                |
| selectstart² | window            | -           | yes                |
| click³       | window            | -           | yes                |
| dblclick     | selection         | *multiple*⁶ | yes                |
| wheel⁸       | selection         | zoom⁷       | yes                |
| touchstart   | selection         | *multiple*⁶ | no⁴                |
| touchmove    | selection         | zoom        | yes                |
| touchend     | selection         | end         | no⁴                |
| touchcancel  | selection         | end         | no⁴                |

The propagation of all consumed events is [immediately stopped](https://dom.spec.whatwg.org/#dom-event-stopimmediatepropagation).

¹ Necessary to capture events outside an iframe; see [d3-drag#9](https://github.com/d3/d3-drag/issues/9).
<br>² Only applies during an active, mouse-based gesture; see [d3-drag#9](https://github.com/d3/d3-drag/issues/9).
<br>³ Only applies immediately after some mouse-based gestures; see [*zoom*.clickDistance](#zoom_clickDistance).
<br>⁴ Necessary to allow [click emulation](https://developer.apple.com/library/ios/documentation/AppleApplications/Reference/SafariWebContent/HandlingEvents/HandlingEvents.html#//apple_ref/doc/uid/TP40006511-SW7) on touch input; see [d3-drag#9](https://github.com/d3/d3-drag/issues/9).
<br>⁵ Ignored if within 500ms of a touch gesture ending; assumes [click emulation](https://developer.apple.com/library/ios/documentation/AppleApplications/Reference/SafariWebContent/HandlingEvents/HandlingEvents.html#//apple_ref/doc/uid/TP40006511-SW7).
<br>⁶ Double-click and double-tap initiate a transition that emits start, zoom and end events; see [*zoom*.tapDistance](#zoom_tapDistance).
<br>⁷ The first wheel event emits a start event; an end event is emitted when no wheel events are received for 150ms.
<br>⁸ Ignored if already at the corresponding limit of the [scale extent](#zoom_scaleExtent).

## zoom() {#zoom}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · Creates a new zoom behavior. The returned behavior, [*zoom*](#_zoom), is both an object and a function, and is typically applied to selected elements via [*selection*.call](../d3-selection/control-flow.md#selection_call).

## *zoom*(*selection*) {#_zoom}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · Applies this zoom behavior to the specified [*selection*](../d3-selection.md), binding the necessary event listeners to allow panning and zooming, and initializing the [zoom transform](./transform.md) on each selected element to the identity transform if not already defined.

This function is typically not invoked directly, and is instead invoked via [*selection*.call](../d3-selection/control-flow.md#selection_call). For example, to instantiate a zoom behavior and apply it to a selection:

```js
selection.call(d3.zoom().on("zoom", zoomed));
```

Internally, the zoom behavior uses [*selection*.on](../d3-selection/events.md#selection_on) to bind the necessary event listeners for zooming. The listeners use the name `.zoom`, so you can subsequently unbind the zoom behavior as follows:

```js
selection.on(".zoom", null);
```

To disable just wheel-driven zooming (say to not interfere with native scrolling), you can remove the zoom behavior’s wheel event listener after applying the zoom behavior to the selection:

```js
selection
    .call(zoom)
    .on("wheel.zoom", null);
```

Alternatively, use [*zoom*.filter](#zoom_filter) for greater control over which events can initiate zoom gestures.

Applying the zoom behavior also sets the [-webkit-tap-highlight-color](https://developer.apple.com/library/mac/documentation/AppleApplications/Reference/SafariWebContent/AdjustingtheTextSize/AdjustingtheTextSize.html#//apple_ref/doc/uid/TP40006510-SW5) style to transparent, disabling the tap highlight on iOS. If you want a different tap highlight color, remove or re-apply this style after applying the drag behavior.

## *zoom*.transform(*selection*, *transform*, *point*) {#zoom_transform}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · If *selection* is a selection, sets the [current zoom transform](./transform.md#zoomTransform) of the selected elements to the specified *transform*, instantaneously emitting start, zoom and end [events](#zoom-events).

If *selection* is a transition, defines a “zoom” tween to the specified *transform* using [interpolateZoom](../d3-interpolate/zoom.md#interpolateZoom), emitting a start event when the transition starts, zoom events for each tick of the transition, and then an end event when the transition ends (or is interrupted). The transition will attempt to minimize the visual movement around the specified *point*; if the *point* is not specified, it defaults to the center of the viewport [extent](#zoom_extent).

The *transform* may be specified either as a [zoom transform](./transform.md) or as a function that returns a zoom transform; similarly, the *point* may be specified either as a two-element array [*x*, *y*] or a function that returns such an array. If a function, it is invoked for each selected element, being passed the current event (`event`) and datum `d`, with the `this` context as the current DOM element.

This function is typically not invoked directly, and is instead invoked via [*selection*.call](../d3-selection/control-flow.md#selection_call) or [*transition*.call](../d3-transition/control-flow.md#transition_call). For example, to reset the zoom transform to the [identity transform](#zoomIdentity) instantaneously:

```js
selection.call(zoom.transform, d3.zoomIdentity);
```

To smoothly reset the zoom transform to the identity transform over 750 milliseconds:

```js
selection.transition().duration(750).call(zoom.transform, d3.zoomIdentity);
```

This method requires that you specify the new zoom transform completely, and does not enforce the defined [scale extent](#zoom_scaleExtent) and [translate extent](#zoom_translateExtent), if any. To derive a new transform from the existing transform, and to enforce the scale and translate extents, see the convenience methods [*zoom*.translateBy](#zoom_translateBy), [*zoom*.scaleBy](#zoom_scaleBy) and [*zoom*.scaleTo](#zoom_scaleTo).

## *zoom*.translateBy(*selection*, *x*, *y*) {#zoom_translateBy}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · If *selection* is a selection, [translates](./transform.md#transform_translate) the [current zoom transform](./transform.md#zoomTransform) of the selected elements by *x* and *y*, such that the new *t<sub>x1</sub>* = *t<sub>x0</sub>* + *kx* and *t<sub>y1</sub>* = *t<sub>y0</sub>* + *ky*. If *selection* is a transition, defines a “zoom” tween translating the current transform. This method is a convenience method for [*zoom*.transform](#zoom_transform). The *x* and *y* translation amounts may be specified either as numbers or as functions that return numbers. If a function, it is invoked for each selected element, being passed the current datum `d` and index `i`, with the `this` context as the current DOM element.

## *zoom*.translateTo(*selection*, *x*, *y*, *p*) {#zoom_translateTo}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · If *selection* is a selection, [translates](./transform.md#transform_translate) the [current zoom transform](./transform.md#zoomTransform) of the selected elements such that the given position ⟨*x*,*y*⟩ appears at given point *p*. The new *t<sub>x</sub>* = *p<sub>x</sub>* - *kx* and *t<sub>y</sub>* = *p<sub>y</sub>* - *ky*. If *p* is not specified, it defaults to the center of the viewport [extent](#zoom_extent). If *selection* is a transition, defines a “zoom” tween translating the current transform. This method is a convenience method for [*zoom*.transform](#zoom_transform). The *x* and *y* coordinates may be specified either as numbers or as functions that returns numbers; similarly the *p* point may be specified either as a two-element array [*p<sub>x</sub>*,*p<sub>y</sub>*] or a function. If a function, it is invoked for each selected element, being passed the current datum `d` and index `i`, with the `this` context as the current DOM element.

## *zoom*.scaleBy(*selection*, *k*, *p*) {#zoom_scaleBy}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · If *selection* is a selection, [scales](./transform.md#transform_scale) the [current zoom transform](./transform.md#zoomTransform) of the selected elements by *k*, such that the new *k₁* = *k₀k*. The reference point *p* does move. If *p* is not specified, it defaults to the center of the viewport [extent](#zoom_extent). If *selection* is a transition, defines a “zoom” tween translating the current transform. This method is a convenience method for [*zoom*.transform](#zoom_transform). The *k* scale factor may be specified either as a number or a function that returns a number; similarly the *p* point may be specified either as a two-element array [*p<sub>x</sub>*,*p<sub>y</sub>*] or a function. If a function, it is invoked for each selected element, being passed the current datum `d` and index `i`, with the `this` context as the current DOM element.

## *zoom*.scaleTo(*selection*, *k*, *p*) {#zoom_scaleTo}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · If *selection* is a selection, [scales](./transform.md#transform_scale) the [current zoom transform](./transform.md#zoomTransform) of the selected elements to *k*, such that the new *k₁* = *k*. The reference point *p* does move. If *p* is not specified, it defaults to the center of the viewport [extent](#zoom_extent). If *selection* is a transition, defines a “zoom” tween translating the current transform. This method is a convenience method for [*zoom*.transform](#zoom_transform). The *k* scale factor may be specified either as a number or a function that returns a number; similarly the *p* point may be specified either as a two-element array [*p<sub>x</sub>*,*p<sub>y</sub>*] or a function. If a function, it is invoked for each selected element, being passed the current datum `d` and index `i`, with the `this` context as the current DOM element.

## *zoom*.constrain(*constrain*) {#zoom_constrain}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · If *constrain* is specified, sets the transform constraint function to the specified function and returns the zoom behavior. If *constrain* is not specified, returns the current constraint function, which defaults to:

```js
function constrain(transform, extent, translateExtent) {
  var dx0 = transform.invertX(extent[0][0]) - translateExtent[0][0],
      dx1 = transform.invertX(extent[1][0]) - translateExtent[1][0],
      dy0 = transform.invertY(extent[0][1]) - translateExtent[0][1],
      dy1 = transform.invertY(extent[1][1]) - translateExtent[1][1];
  return transform.translate(
    dx1 > dx0 ? (dx0 + dx1) / 2 : Math.min(0, dx0) || Math.max(0, dx1),
    dy1 > dy0 ? (dy0 + dy1) / 2 : Math.min(0, dy0) || Math.max(0, dy1)
  );
}
```

The constraint function must return a [*transform*](./transform.md) given the current *transform*, [viewport extent](#zoom_extent) and [translate extent](#zoom_translateExtent). The default implementation attempts to ensure that the viewport extent does not go outside the translate extent.

## *zoom*.filter(*filter*) {#zoom_filter}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · If *filter* is specified, sets the filter to the specified function and returns the zoom behavior. If *filter* is not specified, returns the current filter, which defaults to:

```js
function filter(event) {
  return (!event.ctrlKey || event.type === 'wheel') && !event.button;
}
```

The filter is passed the current event (`event`) and datum `d`, with the `this` context as the current DOM element. If the filter returns falsey, the initiating event is ignored and no zoom gestures are started. Thus, the filter determines which input events are ignored. The default filter ignores mousedown events on secondary buttons, since those buttons are typically intended for other purposes, such as the context menu.

## *zoom*.touchable(*touchable*) {#zoom_touchable}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · If *touchable* is specified, sets the touch support detector to the specified function and returns the zoom behavior. If *touchable* is not specified, returns the current touch support detector, which defaults to:

```js
function touchable() {
  return navigator.maxTouchPoints || ("ontouchstart" in this);
}
```

Touch event listeners are only registered if the detector returns truthy for the corresponding element when the zoom behavior is [applied](#_zoom). The default detector works well for most browsers that are capable of touch input, but not all; Chrome’s mobile device emulator, for example, fails detection.

## *zoom*.wheelDelta(*delta*) {#zoom_wheelDelta}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · If *delta* is specified, sets the wheel delta function to the specified function and returns the zoom behavior. If *delta* is not specified, returns the current wheel delta function, which defaults to:

```js
function wheelDelta(event) {
  return -event.deltaY * (event.deltaMode === 1 ? 0.05 : event.deltaMode ? 1 : 0.002) * (event.ctrlKey ? 10 : 1);
}
```

The value *Δ* returned by the wheel delta function determines the amount of scaling applied in response to a [WheelEvent](https://developer.mozilla.org/en-US/docs/Web/API/WheelEvent). The scale factor [*transform*.k](./transform.md#zoomTransform) is multiplied by 2<sup>*Δ*</sup>; for example, a *Δ* of +1 doubles the scale factor, *Δ* of -1 halves the scale factor.

## *zoom*.extent(*extent*) {#zoom_extent}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · If *extent* is specified, sets the viewport extent to the specified array of points [[*x0*, *y0*], [*x1*, *y1*]], where [*x0*, *y0*] is the top-left corner of the viewport and [*x1*, *y1*] is the bottom-right corner of the viewport, and returns this zoom behavior. The *extent* may also be specified as a function which returns such an array; if a function, it is invoked for each selected element, being passed the current datum `d`, with the `this` context as the current DOM element.

If *extent* is not specified, returns the current extent accessor, which defaults to [[0, 0], [*width*, *height*]] where *width* is the [client width](https://developer.mozilla.org/en-US/docs/Web/API/Element/clientWidth) of the element and *height* is its [client height](https://developer.mozilla.org/en-US/docs/Web/API/Element/clientHeight); for SVG elements, the nearest ancestor SVG element’s viewBox, or [width](https://www.w3.org/TR/SVG/struct.html#SVGElementWidthAttribute) and [height](https://www.w3.org/TR/SVG/struct.html#SVGElementHeightAttribute) attributes, are used. Alternatively, consider using [*element*.getBoundingClientRect](https://developer.mozilla.org/en-US/docs/Web/API/Element/getBoundingClientRect).

The viewport extent affects several functions: the center of the viewport remains fixed during changes by [*zoom*.scaleBy](#zoom_scaleBy) and [*zoom*.scaleTo](#zoom_scaleTo); the viewport center and dimensions affect the path chosen by [interpolateZoom](../d3-interpolate/zoom.md#interpolateZoom); and the viewport extent is needed to enforce the optional [translate extent](#zoom_translateExtent).

## *zoom*.scaleExtent(*extent*) {#zoom_scaleExtent}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · If *extent* is specified, sets the scale extent to the specified array of numbers [*k0*, *k1*] where *k0* is the minimum allowed scale factor and *k1* is the maximum allowed scale factor, and returns this zoom behavior. If *extent* is not specified, returns the current scale extent, which defaults to [0, ∞]. The scale extent restricts zooming in and out. It is enforced on interaction and when using [*zoom*.scaleBy](#zoom_scaleBy), [*zoom*.scaleTo](#zoom_scaleTo) and [*zoom*.translateBy](#zoom_translateBy); however, it is not enforced when using [*zoom*.transform](#zoom_transform) to set the transform explicitly.

If the user tries to zoom by wheeling when already at the corresponding limit of the scale extent, the wheel events will be ignored and not initiate a zoom gesture. This allows the user to scroll down past a zoomable area after zooming in, or to scroll up after zooming out. If you would prefer to always prevent scrolling on wheel input regardless of the scale extent, register a wheel event listener to prevent the browser default behavior:

```js
selection
    .call(zoom)
    .on("wheel", event => event.preventDefault());
```

## *zoom*.translateExtent(*extent*) {#zoom_translateExtent}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · If *extent* is specified, sets the translate extent to the specified array of points [[*x0*, *y0*], [*x1*, *y1*]], where [*x0*, *y0*] is the top-left corner of the world and [*x1*, *y1*] is the bottom-right corner of the world, and returns this zoom behavior. If *extent* is not specified, returns the current translate extent, which defaults to [[-∞, -∞], [+∞, +∞]]. The translate extent restricts panning, and may cause translation on zoom out. It is enforced on interaction and when using [*zoom*.scaleBy](#zoom_scaleBy), [*zoom*.scaleTo](#zoom_scaleTo) and [*zoom*.translateBy](#zoom_translateBy); however, it is not enforced when using [*zoom*.transform](#zoom_transform) to set the transform explicitly.

## *zoom*.clickDistance(*distance*) {#zoom_clickDistance}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · If *distance* is specified, sets the maximum distance that the mouse can move between mousedown and mouseup that will trigger a subsequent click event. If at any point between mousedown and mouseup the mouse is greater than or equal to *distance* from its position on mousedown, the click event following mouseup will be suppressed. If *distance* is not specified, returns the current distance threshold, which defaults to zero. The distance threshold is measured in client coordinates ([*event*.clientX](https://developer.mozilla.org/en-US/docs/Web/API/MouseEvent/clientX) and [*event*.clientY](https://developer.mozilla.org/en-US/docs/Web/API/MouseEvent/clientY)).

## *zoom*.tapDistance(*distance*) {#zoom_tapDistance}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · If *distance* is specified, sets the maximum distance that a double-tap gesture can move between first touchstart and second touchend that will trigger a subsequent double-click event. If *distance* is not specified, returns the current distance threshold, which defaults to 10. The distance threshold is measured in client coordinates ([*event*.clientX](https://developer.mozilla.org/en-US/docs/Web/API/MouseEvent/clientX) and [*event*.clientY](https://developer.mozilla.org/en-US/docs/Web/API/MouseEvent/clientY)).

## *zoom*.duration(*duration*) {#zoom_duration}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · If *duration* is specified, sets the duration for zoom transitions on double-click and double-tap to the specified number of milliseconds and returns the zoom behavior. If *duration* is not specified, returns the current duration, which defaults to 250 milliseconds. If the duration is not greater than zero, double-click and -tap trigger instantaneous changes to the zoom transform rather than initiating smooth transitions.

To disable double-click and double-tap transitions, you can remove the zoom behavior’s dblclick event listener after applying the zoom behavior to the selection:

```js
selection
    .call(zoom)
    .on("dblclick.zoom", null);
```

## *zoom*.interpolate(*interpolate*) {#zoom_interpolate}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · If *interpolate* is specified, sets the interpolation factory for zoom transitions to the specified function. If *interpolate* is not specified, returns the current interpolation factory, which defaults to [interpolateZoom](../d3-interpolate/zoom.md#interpolateZoom) to implement smooth zooming. To apply direct interpolation between two views, try [interpolate](../d3-interpolate/value.md#interpolate) instead.

## *zoom*.on(*typenames*, *listener*) {#zoom_on}

[Source](https://github.com/d3/d3-zoom/blob/main/src/zoom.js) · If *listener* is specified, sets the event *listener* for the specified *typenames* and returns the zoom behavior. If an event listener was already registered for the same type and name, the existing listener is removed before the new listener is added. If *listener* is null, removes the current event listeners for the specified *typenames*, if any. If *listener* is not specified, returns the first currently-assigned listener matching the specified *typenames*, if any. When a specified event is dispatched, each *listener* will be invoked with the same context and arguments as [*selection*.on](../d3-selection/control-flow.md#selection_on) listeners: the current event (`event`) and datum `d`, with the `this` context as the current DOM element.

The *typenames* is a string containing one or more *typename* separated by whitespace. Each *typename* is a *type*, optionally followed by a period (`.`) and a *name*, such as `zoom.foo` and `zoom.bar`; the name allows multiple listeners to be registered for the same *type*. The *type* must be one of the following:

* `start` - after zooming begins (such as on mousedown).
* `zoom` - after a change to the zoom transform (such as on mousemove).
* `end` - after zooming ends (such as on mouseup ).

See [*dispatch*.on](../d3-dispatch.md#dispatch_on) for more.

## Zoom events

When a [zoom event listener](#zoom_on) is invoked, it receives the current zoom event as a first argument. The *event* object exposes several fields:

* *event*.target - the associated [zoom behavior](#zoom).
* *event*.type - the string “start”, “zoom” or “end”; see [*zoom*.on](#zoom_on).
* *event*.transform - the current [zoom transform](./transform.md).
* *event*.sourceEvent - the underlying input event, such as mousemove or touchmove.