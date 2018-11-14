# WebXR Device API - Input
This document is a subsection of the main WebXR Device API explainer document which can be found [here](explainer.md).  The main explainer contains all the information you could possibly want to know about setting up a WebXR session, the render loop, and more.  In contrast, this document covers how to manage input across the range of XR hardware.

## Usage

XR hardware provides a wide variety of input mechanisms, ranging from single state buttons to fully tracked controllers with multiple buttons, joysticks, triggers, or touchpads. While the intent is to eventually support the full range of available hardware, for the initial version of the WebXR Device API the focus is on enabling a more universal "point and click" style system that can be supported in some capacity by any known XR device and in inline mode.

In this model every input source has a ray that indicates what is being pointed at, called the "Target Ray", and reports when the primary action for that device has been triggered, surfaced as a "select" event. When the select event is fired the XR application can use the target ray of the input source that generated the event to determine what the user was attempting to interact with and respond accordingly. Additionally, if the input source represents a tracked device a "Grip" transform will also be provided to indicate where a mesh should be rendered to align with the physical device.

### Enumerating input sources

Calling the `getInputSources()` function on an `XRSession` will return a list of all `XRInputSource`s that the user agent considers active. An `XRInputSource` may represent a tracked controller, inputs built into the headset itself, or more ephemeral input mechanism like tracking of hand gestures. When input sources are added to or removed from the list of available input sources the `inputsourceschange` event will be fired on the `XRSession` object to indicate that any cached copies of the list should be refreshed.

```js
// Get the current list of input sources.
let xrInputSources = xrSession.getInputSources();

// Update the list of input sources if it ever changes.
xrSession.addEventListener('inputsourceschange', (ev) => {
  xrInputSources = xrSession.getInputSources();
});
```

The properties of an XRInputSource object are immutable. If a device can be manipulated in such a way that these properties can change, the `XRInputSource` will be removed and recreated.

### Input poses

Each input source provides two `XRSpace`s, which can be used to query an `XRPose` using the `getPose()` function of any `XRFrame`. Getting the pose requires passing in the `XRSpace` you want the pose for, as well as the `XRSpace` the returned pose should be relative to (which may be an `XRReferenceSpace`). Just like `getViewerPose()`, `getPose()` may return `null` in cases where tracking has been lost, or the `XRSpace`'s `XRInputSource` instance is no longer connected or available.

The `gripSpace` represents a space where if the user was holding a straight rod in their hand it would be aligned with the negative Z axis (forward) and the origin rests at their palm. This enables developers to properly render a virtual object held in the user's hand. For example, a sword would be positioned so that the blade points directly down the negative Z axis and the center of the handle is at the origin.

If the input source has only 3DOF, the grip pose may represent only a translation or rotation based on tracking capability. An example of this case is for physical hands on some AR devices which only have a tracked position. The `gripSpace` will be `null` if the input source isn't trackable.

An input source will also provide its preferred pointing ray, given by the `XRInputSource`'s `targetRaySpace`. This can be used to get the `XRPose` for the input's target rays.

An `XRRay` can be constructed from the `XRPose`'s `transform`. A ray includes both an `origin` and `direction`, both given as `DOMPointReadOnly`s. The `origin` represents a 3D coordinate in space with a `w` component that must be 1, and the `direction` represents a normalized 3D directional vector with a `w` component that must be 0. The `XRRay`'s `matrix` represents the transform from a ray originating at `[0, 0, 0]` and extending down the negative Z axis to the ray described by the `XRRay`'s `origin` and `direction`. This is useful for positioning graphical representations of the ray.

The `targetRaySpace` will never be `null`. It's value will differ based on the type of input source that produces it, which is represented by the `targetRayMode` attribute:

  * `'gaze'` indicates the target ray will originate at the user's head and follow the direction they are looking (this is commonly referred to as a "gaze input" device). While it may be possible for these devices to be tracked (and have a `gripSpace`), the head gaze is used for targeting. Example devices: 0DOF clicker, regular gamepad, voice command, tracked hands.
  * `'tracked-pointer'` indicates that the target ray originates from either a handheld device or other hand-tracking mechanism and represents that the user is using their hands or the held device for pointing. The exact orientation of the ray relative to a given device should follow platform-specific guidelines if there are any. In the absence of platform-specific guidance or a physical device, the target ray should most likely point in the same direction as the user's index finger if it was outstretched.
  * `'screen'` indicates that the input source was an interaction with a session's output context canvas element, such as a mouse click or touch event. Only applicable for inline sessions or an immersive AR session being displayed on a 2D screen. See [Screen Input](#screen_input) for more details.

```js
// Loop over every input source and get their pose for the current frame.
for (let inputSource of xrInputSources) {
  let targetRayPose = xrFrame.getPose(inputSource.targetRaySpace, xrReferenceSpace);

  // Check to see if the pose is valid
  if (targetRayPose) {
    // Highlight any objects that the target ray intersects with.
    let hoveredObject = scene.getObjectIntersectingRay(new XRRay(targetRayPose.transform));
    if (hoveredObject) {
      // Render a visualization of the object that is highlighted (see below).
      drawHighlightFrom(hoveredObject, inputSource);
    }
  }

  // Render a visualization of the input source if appropriate.
  // (See next section for details).
  renderInputSource(xrFrame, inputSource);
}
```

Some platforms may support both tracked and non-tracked input sources concurrently (such as a pair of `'tracked-pointer'` 6DOF controllers plus a regular `'gaze'` clicker). Since `xrSession.getInputSources()` returns all connected input sources, an application should take into consideration the most recently used input sources when rendering UI hints, such as a cursor, ray or highlight.

```js
// Keep track of the last-used input source
var lastInputSource = null;

function onSessionStarted(session) {
  session.addEventListener("selectstart", event => {
    // Update the last-used input source
    lastInputSource = event.inputSource;
  });
  session.addEventListener("inputsourceschange", ev => {
    // Choose an appropriate default from available inputSources, such as prioritizing based on the value of targetRayMode:
    // 'screen' over 'tracked-pointer' over 'gaze'.
    lastInputSource = computePreferredInputSource(session.getInputSources());
  });

  // Remainder of session initialization logic.
}

function drawHighlightFrom(hoveredObject, inputSource) {
  // Only highlight meshes that are targeted by the last used input source.
  if (inputSource == lastInputSource) {
    // Render a visualization of the highlighted object. (see next section)
    renderer.drawHighlightFrom(hoveredObject);
  }
}

// Called by the fictional app/middleware
function drawScene() {
  // Display only a single cursor or ray, on the most recently used input source.
  if (lastInputSource) {
    let targetRayPose = xrFrame.getPose(lastInputSource.targetRaySpace, xrReferenceSpace);
    if (targetRayPose) {
      // Render a visualization of the target ray/cursor of the active input source. (see next section)
      renderCursor(lastInputSource, targetRayPose)
    }
  }
}
```

### Selection event handling

The initial version of the WebXR Device API spec is limited to only recognizing when an input source's primary action has occurred. The primary action differs based on the hardware, and may indicate (but is not limited to):

  * Pressing a trigger
  * Clicking a touchpad
  * Tapping a button
  * Making a hand gesture
  * Speaking a command
  * Clicking or touching a canvas.

Three events are fired on the `XRSession` related to these primary actions: `selectstart`, `selectend`, and `select`.

A `selectstart` event indicates that the primary action has been initiated. It will most commonly be associated with pressing a button or trigger.

A `selectend` event indicates that the primary action has ended. It will most commonly be associated with releasing a button or trigger. A `selectend` event must also be fired if the input source is disconnected after a primary action has been initiated, or the primary action has otherwise been cancelled. In that case an associated `select` event will not be fired.

A `select` event indicates that a primary action has been completed. `select` events are considered to be [triggered by user activation](https://html.spec.whatwg.org/multipage/interaction.html#triggered-by-user-activation) and as such can be used to begin playing media or other trusted interactions.

For primary actions that are instantaneous without a clear start and end point (such as a verbal command), all three events should still fire in the sequence `selectstart`, `selectend`, `select`.

All three events are `XRInputSourceEvent` events. When fired the event's `inputSource` attribute must contain the `XRInputSource` that produced the event. The event's `frame` attribute must contain a valid `XRFrame` that can be used to query the input and device poses at the time the selection event occurred. The `XRViewerPose`'s `views` array must be empty.

In most cases applications will only need to listen for the `select` event for basic interactions like clicking on buttons.

```js
function onSessionStarted(session) {
  session.addEventListener("select", onSelect);

  // Remainder of session initialization logic.
}

function onSelect(event) {
  let targetRayPose = event.frame.getPose(event.inputSource.targetRaySpace, xrReferenceSpace);
  if (targetRayPose) {
    // Ray cast into scene to determine if anything was hit.
    let selectedObject = scene.getObjectIntersectingRay(new XRRay(targetRayPose.transform));
    if (selectedObject) {
      selectedObject.onSelect();
    }
  }
}
```

Some input sources (such as those with a `targetRayMode` of `screen`) will be only be added to the list of input sources whilst a primary action is occurring. In these cases, the `inputsourceschange` event will fire just prior to the `selectstart` event, then again when the input source is removed after the `selectend` event.

`selectstart` and `selectend` can be useful for handling dragging, painting, or other continuous motions.

In some cases tracked input sources cannot accurately track their position in space, and provides an estimated position based on the sensor data available to it. This is the case, for example, for the Daydream and GearVR 3DoF controllers, which use an arm model to approximate controller position based on rotation. In these cases the `emulatedPosition` attribute of the `XRInputPose` should be set to `true` to indicate that the translation components of the pose matrices may not be accurate.

While most applications will wish to use a targeting ray from the input source pose, it is possible to support only gaze and commit interactions such that the targeting ray always matches the head pose even if trackable controllers are connected. In this case, the `select` event should still be used to handle interaction events, but the device pose can be used to create the targeting ray.

```js
function onSelect(event) {
  // Use the viewer pose to create a ray from the head, regardless of whether controllers are connected.
  let viewerPose = event.frame.getViewerPose(xrReferenceSpace);

  // Ray cast into scene with the viewer pose to determine if anything was hit.
  // Assumes the use of a fictionalized math and scene library.
  let selectedObject = scene.getObjectIntersectingRay(new XRRay(viewerPose.transform));
  if (selectedObject) {
    selectedObject.onSelect();
  }
}
```

### Rendering input sources

Most applications will want to visually represent the input sources somehow. The appropriate type of visualization to be used depends on the value of the `targetRayMode` attribute:

  * `'gaze'`: A cursor should be drawn at some distance down the target ray, ideally at the depth of the first surface it intersects with, so the user can identify what will be interacted with when a select event is fired. It's not appropriate to draw a controller or ray in this case, since they may obscure the user's vision or be difficult to visually converge on.
  * `'tracked-pointer'`: If the `gripSpace` is not `null` an application-appropriate controller model should be drawn using that transform. If appropriate for the experience, the a visualization of the target ray and a cursor as described in the `'gaze'` should also be drawn.
  * `'screen'`: In all cases the point of origin of the target ray is obvious and no visualization is needed.

Determining what to render is important as well. In some cases it's appropriate for the application to render a contextually appropriate model (such as a racket in a tennis game), in which case the physical appearance of the input source doesn't matter much. Many other times, though, it's desirable for the application to display a device that matches what the user is holding, especially when relaying instructions about it's use. Finally there are times when it's best to not render anything at all, such as when the XR device uses a transparent display and the user can see their hands and/or any tracked devices without app assistance.

Controller devices should generally only be rendered if the `XRSession`'s `environmentBlendMode` is `'opaque'`, as any other mode implies that the user can see any physical device they may be holding. Rendering cursors or target rays may still be desirable.

If the input source is a handheld device, the `XRInputSource` object should have a non-null `gamepad` attribute. The `Gamepad`'s `id` is used to determine what should be rendered if the app intends to visualize the input source itself, rather than an alternative virtual object. (See the section on [Button and Axis State](#button-and-axis-state) for more details.)

The WebXR Device API currently does not offer any way to retrieve renderable resources that represent the input devices from the API itself, and as such the `Gamepad`'s `id` must be used as a key to load an appropriate resources from the application's server or a CDN. The example below presumes that the `getControllerMesh` call would do the required lookup and caching.

```js
// These methods presumes the use of a fictionalized rendering library.

// Render a visualization of the input source - eg. a controller mesh.
function renderInputSource(xrFrame, inputSource) {
  // Don't render an input source if it doesn't have a grip space, doesn't have
  // a gamepad, or if the blend mode is not 'opaque'. 
  if (!inputSource.gripSpace || !inputSource.gamepad ||
      xrFrame.session.environmentBlendMode != 'opaque')
    return;

  // Retrieve a mesh to render based on the gamepad object's id. If no mesh is
  // found skip rendering.
  let controllerMesh = getControllerMesh(inputSource.gamepad.id);
  if (!controllerMesh)
    return;

  // Only render a controller mesh if we have a valid gripPose.
  let gripPose = xrFrame.getPose(inputSource.gripSpace, xrFrameOfRef);
  if (gripPose) {
    renderer.drawMeshAtTransform(controllerMesh, gripPose.transform);
  }
}

// Render a visualization of target ray of the input source - eg. a line or cursor.
// Presumes the use of a fictionalized rendering library.
function renderCursor(inputSource, targetRayPose) {
  // Only render a target ray if this was the most recently used input source.
  if (inputSource.targetRayMode == "tracked-pointer") {
    // Draw targeting rays for tracked-pointer devices only.
    renderer.drawRay(new XRRay(targetRayPose.transform));
  }

  if (inputSource.targetRayMode != 'screen') {
    // Draw a cursor for gazing and tracked-pointer devices only.
    let cursorPosition = scene.getIntersectionPoint(new XRRay(targetRayPose.transform));
    if (cursorPosition) {
      renderer.drawCursor(cursorPosition);
    }
  }
}
```

### Grabbing and dragging

While the primary motivation of this input model is a compatible "target and click" interface, more complex interactions such as grabbing and dragging with input sources can also be achieved using only the `select` events.

```js
// Stores details of an active drag interaction, is any
let activeDragInteraction = null;

function onSessionStarted(session) {
  session.addEventListener('selectstart', onSelectStart);
  session.addEventListener('selectend', onSelectEnd);

  // Remainder of session initialization logic.
}

function onSelectStart(event) {
  // Ignore the event if we are already dragging
  if (activeDragInteraction)
    return;

  let targetRayPose = event.frame.getPose(event.inputSource.targetRaySpace, xrReferenceSpace);

  // Use the input source target ray to find a draggable object in the scene
  let hitResult = scene.hitTest(new XRRay(targetRayPose.transform));
  if (hitResult && hitResult.draggable) {
    // Use the targetRayPose position to drag the intersected object.
    activeDragInteraction = {
      target: hitResult,
      targetStartPosition: hitResult.position,
      inputSource: event.inputSource,
      inputSourceStartPosition: targetRayPose.transform.position;
    };
  }
}

// Only end the drag when the input source that started dragging releases the select action
function onSelectEnd(event) {
  if (activeDragInteraction && event.inputSource == activeDragInteraction.inputSource)
    activeDragInteraction = null;
}

// Called by the fictional app/middleware every frame
function onUpdateScene() {
  if (activeDragInteraction) {
    let targetRayPose = frame.getPose(activeDragInteraction.inputSource.targetRaySpace, xrReferenceSpace);
    if (targetRayPose) {
      // Determine the vector from the start of the drag to the input source's current position
      // and position the draggable object accordingly
      let deltaPosition = Vector3.subtract(targetRayPose.transform.position, activeDragInteraction.inputSourceStartPosition);
      let newPosition = Vector3.add(activeDragInteraction.targetStartPosition, deltaPosition);
      activeDragInteraction.target.setPosition(newPosition);
    }
  }
}
```

### Screen Input

When using an inline session or an immersive AR session on a 2D screen, pointer events on the canvas that created the session's `outputContext` are monitored. `XRInputSource`s are generated in response to allow unified input handling with immersive mode controller or gaze input.

When the canvas receives a `pointerdown` event an `XRInputSource` is created with a `targetRayMode` of `'screen'` and added to the array returned by `getInputSources()`. A `selectstart` event is then fired on the session with the new `XRInputSource`. The `XRInputSource`'s target ray should be updated with every `pointermove` event the canvas receives until a `pointerup` event is received. A `selectend` event is then fired on the session and the `XRInputSource` is removed from the array returned by `getInputSources()`. When the canvas receives a `click` event a `select` event is fired on the session with the appropriate `XRInputSource`.

For each of these events the `XRInputSource`'s target ray must be updated to originate at the point that was interacted with on the canvas, projected onto the near clipping plane (defined by the `depthNear` attribute of the `XRSession`) and extending out into the scene along that projected vector.

## Button and Axis State

Some applications need more than point-and-click style interaction provided by the `select` events. For devices that make use of controllers with buttons and axes, more complete information about the state of those inputs can be observed via the `XRInputSource`'s `gamepad` attribute.

If the `XRInputState` represents an XR controller device with buttons or axes the `gamepad` attribute is an instance of a [`Gamepad`](https://w3c.github.io/gamepad/#gamepad-interface) interface, and `null` otherwise. Examples of controllers that may expose their state this way include Oculus Touch, Vive wands, Oculus Go and Daydream controllers, or other similar devices. Controllers not directly associated with the XR device, such as the majority of traditional gamepads, should not be exposed using this interface. Tracked devices without discreet inputs, such as optical hand tracking, should also have a `null` `gamepad` attribute.

`Gamepad` instances reported in this way have several notable behavioral changes vs. the ones reported by `navigator.getGamepads()`:

  - `Gamepad` instances connected to an `XRInputSource` must not be included in the array returned by `navigator.getGamepads()`.
  - The `Gamepad`'s `index` attribute must be `-1`.
  - The `Gamepad`'s `connected` attribute must always be `true`.
  - The `Gamepad`'s `mapping` attribute must be `xr-standard`.

Finally, the `id` attribute for `Gamepad`s surfaced by the WebXR API are more strictly formatted than those of traditional gamepads in order to make them a more appropriate key for determining rendering assets.

  - The `id` MAY be `'unknown'` if the type of input source cannot be reliably identified or the UA determines that the input source type must be masked for any reason. Applications should render a generic input device in this case.
  - Inline sessions MUST only expose `id`s of `'unknown'`.
  - Otherwise the `id` should be a lower-case string that describes the physical input source.
    - For most devices this SHOULD be of the format `<vendor>-<product-id>`. For example: `oculus-touch`. UAs SHOULD make an effort to align on the strings that are returned for any given device.
    - It should not include an indication of the handedness of the input source, as that should be implied by the `handedness` attribute.

All other attributes behave as described in the [Gamepad](https://w3c.github.io/gamepad/) specification.

The WebXR Device API introduces a new standard controller layout indicated by the `mapping` value of `xr-standard`. (Additional mapping variants may be added in the future if necessary.) This defines a specific layout for the inputs most commonly found on XR controller devices today. The following table describes the buttons/axes and their physical locations:

| Button/Axis  | Location                          |
| ------------ | --------------------------------- |
| buttons[0]   | Primary trigger                   |
| buttons[1]   | Primary Touchpad/Joystick click   |
| buttons[2]   | Grip/Secondary trigger            |
| buttons[3]   | Secondary Touchpad/Joystick click |
| axes[0]      | Primary Touchpad/Joystick X       |
| axes[1]      | Primary Touchpad/Joystick Y       |
| axes[2]      | Secondary Touchpad/Joystick X     |
| axes[3]      | Secondary Touchpad/Joystick Y     |

Additional device-specific inputs may be exposed after these reserved indices, but devices that lack one of the canonical inputs must still preserve their place in the array. If a device has both a touchpad and a joystick the UA should designate one of them to be the primary axis-based input and expose the other at axes[2] and axes[3] with an associated button at button[3].

```js
function onXRFrame(timestamp, frame) {
  let inputSource = primaryInputSource;

  // Check to see if the input source has buttons/axes and is using the standard
  // mapping.
  if (inputSource && inputSource.gamepad &&
      inputSource.gamepad.mapping == 'xr-standard') {
    let gamepad = inputSource.gamepad;
    
    // Use joystick or touchpad values for movement.
    MoveUser(gamepad.axes[0], gamepad.axes[1]);

    // Use the trigger or joystick/touchpad click for painting.
    if (gamepad.buttons[0].pressed || gamepad.buttons[1].pressed) {
      EmitPaint();
    }
    
    // etc.
  }

  // Do the rest of typical frame processing...
}
```

If the application includes interactions that require user activation (such as starting media playback), the application can listen to the `XRInputSource`s `select` events, which fire for every pressed and released button on the controller. When triggered by a controller input, the `XRInputSourceEvent` will include a `buttonIndex` other than `-1` to indicate which button on the gamepad triggered the event.

The UA may update the `gamepad` state at any point, but it must remain constant while running a batch of `XRSession` `requestAnimationFrame` callbacks or event callbacks which provide an `XRFrame`.

### Exposing gamepad values with an action maps

Some native APIs rely on what's commonly referred to as an "action mapping" system to handle controller input. In action map systems the developer creates a list of application-specific actions (such as "undo" or "jump") and suggested input bindings (like "left hand touchpad") that should trigger the related action. Such systems may allow users to re-bind the inputs associated with each action, and may not provide a mechanism for enumerating or monitoring the inputs outside of the action map.

When using an API that limits reading controller input to use of an action map, it is suggested that a mapping be created with one action per possible input, given the same name as the target input. For example, an similar mapping to the following may be used for each device:

| Button/Axis | Action name      | Sample binding            |
|-------------|------------------|---------------------------|
| button[0]   | "trigger"        | "[device]/trigger"        |
| button[1]   | "touchpad-click" | "[device]/touchpad/click" |
| button[2]   | "grip"           | "[device]/grip"           |
| axis[0]     | "touchpad-x"     | "[device]/touchpad/x"     |
| axis[1]     | "touchpad-y"     | "[device]/touchpad/y"     |

If the API does not provided a way to enumerate the available input devices, the UA should provide bindings for the left and right hand instead of a specific device and expose a `Gamepad` for any hand that has at least one non-`null` input.

The UA should not make any attempt to circumvent user remapping of the inputs.

## Appendix A: Proposed partial IDL
This is a partial IDL and is considered additive to the core IDL found in the main [explainer](explainer.md).
```webidl
//
// Session
//

partial interface XRSession {
  FrozenArray<XRInputSource> getInputSources();
  
  attribute EventHandler onselect;
  attribute EventHandler onselectstart;
  attribute EventHandler onselectend;
  attribute EventHandler oninputsourceschange;
};

//
// Frame
//

partial interface XRFrame {
  // Also listed in the spatial-tracking-explainer.md
  XRViewerPose? getViewerPose(optional XRReferenceSpace referenceSpace);
  XRPose? getPose(XRSpace space, XRSpace relativeTo);
};

//
// Input
//

[SecureContext, Exposed=Window,
 Constructor(optional DOMPointInit origin, optional DOMPointInit direction),
 Constructor(XRRigidTransform transform)]
interface XRRay {
  readonly attribute DOMPointReadOnly origin;
  readonly attribute DOMPointReadOnly direction;
  readonly attribute Float32Array matrix;
};

enum XRHandedness {
  "",
  "left",
  "right"
};

enum XRTargetRayMode {
  "gaze",
  "tracked-pointer",
  "screen"
};

[SecureContext, Exposed=Window]
interface XRInputSource {
  readonly attribute XRHandedness handedness;
  readonly attribute XRTargetRayMode targetRayMode;
  readonly attribute XRSpace targetRaySpace;
  readonly attribute XRSpace? gripSpace;
  readonly attribute Gamepad? gamepad;
};

//
// Events
//

[SecureContext, Exposed=Window, Constructor(DOMString type, XRSessionEventInit eventInitDict)]
interface XRSessionEvent : Event {
  readonly attribute XRSession session;
};

dictionary XRSessionEventInit : EventInit {
  required XRSession session;
};

[SecureContext, Exposed=Window,
 Constructor(DOMString type, XRInputSourceEventInit eventInitDict)]
interface XRInputSourceEvent : Event {
  readonly attribute XRFrame frame;
  readonly attribute XRInputSource inputSource;
  readonly attribute long buttonIndex;
};

dictionary XRInputSourceEventInit : EventInit {
  required XRFrame frame;
  required XRInputSource inputSource;
  long buttonIndex = -1;
};
```