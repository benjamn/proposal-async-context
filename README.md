# Async Tasks Tracking for JavaScript

Status: This proposal has not been presented to TC39 yet.

# Motivation

Provide a mechanism to ergonomically track async tasks in JavaScript. There are multiple implementations
in different platforms like `async_hooks` in Node.js and `zones.js` in Angular that provides async task
tracking. These modules works well on their own platform/impl, however they are not in quite same with
each other. Library owners have to adopt both two, or more, to keep a persistent async context across
async tasks execution.

Tracked async tasks are useful for debugging, testing, and profiling. With async tasks tracked, we can
determine what tasks have been scheduled during a specific sync run, and do something additional on
schedule changes, e.g. asserting there is no outstanding async task on end of a test case. Except for
merely tasks tracking, it is also critical to have persist async locals that will be propagated along
with the async task chains, which additional datum can be stored in and fetched from without awareness
or change of the task original code, e.g. `AsyncLocalStorage` in Node.js.

While monkey-patching is quite straightforward solution to track async tasks, there is no way to patch
mechanism like async/await. Also, monkey-patching only works if all third-party libraries with custom
scheduling call a corresponding task awareness registration like `Zone.run`/`AsyncResource.runInAsyncScope`.
Furthermore, for those custom scheduling third-party libraries, we need to get library owners to think in
terms of async context propagation.

In a summary, we would like to have an async task tracking specification right in ECMAScript to be in place
for platform environments to take advantage of it, and a standard JavaScript API to enable third-party
libraries to work on different environments seamlessly.

Priorities (not necessarily in order):
1. **Must** be able to automatically link continuous async tasks.
1. **Must** expose visibility into the async task scheduling and processing.
    1. **Must** not collide or introduce implicit behavior on multiple tracking instance on same async task chain.
    1. **Should** be able to be scoped to the an async task chain.
1. **Must** provide a way to enable logical reentrancy.

Non-goals:
1. Error handling & bubbling through async stacks.
2. Async task interception: this can cause confusion if some imported library can take application owner
unaware actions to change how the application code running pattern. If there are multiple tracking instance
on same async task chain,interception can cause collision and implicit behavior if these instances do not
cooperate well. Thus at this very initial proposal, we'd like to stand away with this feature.

---

Zones are meant to help with the problems of tracking asynchronous code. They are designed as a primitive for
context propagation across multiple logically-connected async operations. As a simple example, consider the
following code:

```js
window.onload = e => {
  // (1)

  fetch("https://example.com").then(res => {
    // (2)

    return processBody(res.body).then(data => {
      // (5)

      const dialog = html`<dialog>Here's some cool data: ${data}
                          <button>OK, cool</button></dialog>`;
      dialog.show();

      dialog.querySelector("button").onclick = () => {
        // (6)
        dialog.close();
      };
    });
  });
};

function processBody(body) {
  // (3)
  return body.json().then(obj => {
    // (4)
    return obj.data;
  });
}
```

At all six marked points, the "async context" is the same: we're in an "async stack" originating from the
`load` event on `window`. Note how `(3)` and `(4)` are outside the lexical context, but is still part of the
same "async stack". And note how the promise chain does not suffice to capture this notion of async stack, as
shown by `(6)`.

Zones are meant specifically as a building block to reify this notion of "logical async context".
However, in this proposal, we are not manipulating of the logical concept in zones, but a side router
to monitor what happened around the async context changes. On top of this, in this proposal, and other work,
perhaps outside of JavaScript, can build on this base association. Such work can accomplish things like:

- Associating "async local data" with the zone, analogous to thread-local storage in other languages, which is accessible to any async operation inside the zone.
- Automatically tracking outstanding async operations within a given zone, to perform cleanup or rendering or test assertion steps afterwards.
- Timing the total time spent in a zone, for analytics or in-the-field profiling.

# Possible Solution

```js
class Zone {
  constructor(zoneSpec, initialStoreGetter);

  attach(): this;
  detach(): this;

  inEffectiveZone(): boolean;

  getStore(): any;
}

interface ZoneSpec {
  scheduledAsyncTask(task);
  beforeAsyncTaskExecute(task);
  afterAsyncTaskExecute(task);
}
```

<!--
TODO: how do we determine a task is not going to be used anymore?

Fundamentally if an object is going to be finalized, it can not be used afterward.
If an async task says it is disposed, `runInAsyncScope` throws once disposed.
-->

For library owners, `AsyncTask`s are preferred to indicate new async tasks' schedule.

```js
class AsyncTask {
  static scheduleAsyncTask(name): AsyncTask;

  get name;

  runInAsyncScope(callback[, thisArg, ...args]);
  [@@dispose]();
}
```

### Using `Zone` for async local storage

<!--
TODO: what's the recommended pattern to enter/attach a zone?

Since async local storage is namespaced in the example: we don't have a global zones effective by default.
Users of async local storage have to declare their own store with their own zones.
Async pattern does work, yet sync one can be adopt more seamlessly to existing codes.
-->

```js
// tracker.js

const zone = Zone(
  /** initialValueGetter */() => ({ startTime: Date.now() }),
);
export function start() {
  // (a)
  zone.attach();
}
export function end() {
  // (b)
  const dur = Date.now() - zone.getStore().startTime;
  console.log('onload duration:', dur);
  zone.detach();
}
```

```js
import * as tracker from './tracker.js'

window.onload = e => {
  // (1)
  tracker.start()

  fetch("https://example.com").then(res => {
    // (2)

    return processBody(res.body).then(data => {
      // (3)

      const dialog = html`<dialog>Here's some cool data: ${data}
                          <button>OK, cool</button></dialog>`;
      dialog.show();

      tracker.end();
    });
  });
};
```

In the example above, `trackStart` and `trackEnd` don't share same lexical scope with actual code functions,
and they are capable of reentrance thus capable of concurrent multi-tracking.

Although `zone.attach` is a sync operation, it is not shared automatically and globally and doesn't have any
side effects on other modules.

# Prior Arts

## Zones
Zones proposed a `Zone` object, which has the following API:

```js
class Zone {
  constructor({ name, parent });

  name;
  get parent();

  fork({ name });
  run(callback);
  wrap(callback);

  static get current();
}
```

Zones have an optional `name`, which is used for tooling and debugging purposes.

Zones can be `fork`ed, creating a _child zone_ whose `parent` pointer is the forker.

The concept of the _current zone_, reified as `Zone.current`, is crucial. Both `run` and `wrap` are designed to manage running the current zone:

- `z.run(callback)` will set the current zone to `z` for the duration of `callback`, resetting it to its previous value afterward. This is how you "enter" a zone.
- `z.wrap(callback)` produces a new function that essentially performs `z.run(callback)` (passing along arguments and this, of course).

The _current zone_ is the async context that propagates with all our operations. In our above example, sites `(1)` through `(6)` would all have the same value of `Zone.current`. If a developer had done something like:

```js
const loadZone = Zone.current.fork({ name: "loading zone" });
window.onload = loadZone.wrap(e => { ... });
```

then at all those sites, `Zone.current` would be equal to `loadZone`.

## Node.js `domain` module

Domain's global central active domain can be consumed by multiple endpoints and be exchanged in any time with
synchronous operation (`domain.enter()`). Since it is possible that some third party module changed active domain on the fly and application owner may unaware of such change, this can introduce unexpected implicit behavior and made domain diagnosis hard.

Check out [Domain Module Postmortem](https://nodejs.org/en/docs/guides/domain-postmortem/) for more details.

## Node.js `async_hooks`

This is what the proposal evolved from. `async_hooks` in Node.js enabled async resources tracking for APM vendors. On which Node.js also implemented `AsyncLocalStorage`.
