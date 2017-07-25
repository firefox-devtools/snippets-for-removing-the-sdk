# The "How To" guide for a SDK-free codebase

A list of snippets and advice for removing SDK bits of code in our codebase, in response to https://bugzilla.mozilla.org/show_bug.cgi?id=1350645

## Replacing sdk/core/heritage ([bug 1368939](https://bugzil.la/1368939))

* [SDK Heritage API reference](https://developer.mozilla.org/en-US/Add-ons/SDK/Low-Level_APIs/core_heritage)
* See [How to deal with decorators](#how-to-deal-with-decorators) for edge cases.

### How to replace heritage’s `Class`

Replace:

```js
var { Class } = require("sdk/core/heritage");
var Dog = Class({
  initialize: function initialize(name) {
    this.name = name;
  },
  type: "dog",
  bark: function bark() {
    return "Ruff! Ruff!"
  }
});
```

With:
```js
class Dog {
  constructor(name) {
    this.name = name;
    this.type = "dog";
  }

  bark() {
    Return "Ruff! Ruff!";
  }
}
```

### How to replace the `Class`' extends

> #### Important
> Start the refactoring always from the superclasses. It means, in the example below, that `Dog` needs to be replaced with ES6 `class` before start to work on `Puppy`.

Replace:
```js
var Puppy = Class({
  extends: Dog,
  initialize: function initialize(breed, name) {
    Dog.prototype.initialize.call(this, name);
    this.breed = breed;
  },
  call: function call(name) {
    return this.name === name ? this.bark() : "";
  }
});
```

With:
```js
class Puppy extends Dog {
  constructor(breed, name) {
    super(name);
    this.breed = breed;
  }
  call(name) {
    return this.name === name ? this.bark() : "";
  }
}
```

### How to replace heritage's `extend`

Use ES6 classes when it's possible, that means also when `extend` is used.

#### Preferable way

Replace:
```js
function Puppy(breed, name) {
  Dog.call(this, name);
  this.breed = breed;
}
Puppy.prototype = extend(Dog.prototype, {
  call: function call(name) {
    return this.name === name ? this.bark() : "";
  }
});
```

With:
```js
class Puppy extends Dog {
  constructor(breed, name) {
    super(name);
    this.breed = breed;
  }
  call(name) {
    return this.name === name ? this.bark() : "";
  }
}
```

It might be not possible everywhere, maybe because [Decorators](#how-to-deal-with-decorators), or because the refactoring of the code would be more complex because object composition. In such cases:

#### Alternative way

Replace:
```js
const { extend } = require("sdk/core/heritage");
```

With:
```js
const { extend } = require("devtools/shared/extend");
```

### How to deal with decorators

Decorators are easy to use in ES5 but they're not possible in ES6. There is a spec for ES7 (see [bug 1368316](https://bugzil.la/1368316)).

1. If the decorator is applied to a `Class`' method, convert the `Class` approach into an `extend` approach.

Replace:
```js
var Puppy = Class({
  extends: Dog,
  initialize: function initialize(breed, name) {
    Dog.prototype.initialize.call(this, name);
    this.breed = breed;
  },
  // `chainable` is a decorator for `call` function, makes
  // the function returns the instance itself.
  call: chainable(function call(name) {
    return this.name === name ? this.bark() : "";
  })
});
```

With:
```js
function Puppy(breed, name) {
  Dog.call(this, name);
  this.breed = breed;
}
Puppy.prototype = extend(Dog.prototype, {
  call: chainable(function call(name) {
    return this.name === name ? this.bark() : "";
  })
});
```

2.  If `extend` already used as above, leave as is.

## Events

In [bug 1381542](https://bugzil.la/1381542) we're unifying the DevTools `EventEmitter` with the SDK's events paradigm.

### Differences with the old `EventEmitter`

#### Major differences

1. `emit` doesn't pass the event type anymore
    ```js
    const EventEmitter = require("devtools/shared/event-emitter");
    const eventBus = new EventEmitter();
    
    eventBus.on("data", console.log);
    // logs "a", "b", "c"
    eventBus.emit("data", "a", "b", "c");
    ```
2.  Now an exception is raised if a bad listener (not a function) is passed to `on` and `once`.

3.  `this` in listeners points to the `EventEmitter` instance (no need to `bind` anymore on simple cases):
    ```js
    const EventEmitter = require("devtools/shared/event-emitter");
    const { emit } = EventEmitter;

    const eventBus = new EventEmitter();

    eventBus.on("data-received", function onDataReceived() {
      console.log(this === eventBus); // true
    });
    ```

#### Additional features

1. It has `on`, `off`, `emit` and `once` as static methods too:
    ```js
    const EventEmitter = require("devtools/shared/event-emitter");
    const { emit } = EventEmitter;

    const eventBus = new EventEmitter();

    eventBus.on("data-received", onDataReceived);

    // This is equivalent to `eventBus.emit`
    emit(eventBus, "data-received");
    ```

2. It has `count` static method
   ```js
    const { count } = require("devtools/shared/event-emitter");
    const eventBus = new EventEmitter();

    eventBus.on("data-received", onDataReceived);

    console.log(count(eventBus, "data-received")); // 1
    console.log(count(eventBus, "foo")); // 0
    ```

3. `off` method can remove all listeners at once
   ```js
    const EventEmitter = require("devtools/shared/event-emitter");
    const eventBus = new EventEmitter();

    eventBus.on("data-received", onDataReceived);
    eventBus.on("data-received", onDataReceivedNotification);
    eventBus.on("data-stop", onDataStop);
    eventBus.on("data-stop", onDataStopNotification);

    // removes only `onDataReceived` from the listeners of "data-received" event
    eventBus.off("data-received", onDataReceived);
    
    // removes all the listener from the listeners of "data-stop"
    eventBus.off("data-stop");
    
    // removes all the listeners
    eventBus.off();
    ```

#### Experimental features
> **Note** those are features that are currently in the [EventEmitter repo](https://github.com/zer0/gecko/tree/event-emitter-1381542), they can be used, but I'd like to see how much sense they have once we fixed the tests.

1. `once` can now have a _predicate_ function as argument.
This was done since I found this pattern often in some tests (actual code below):
```js
function waitForUpdate(inspector, waitForSelectionUpdate) {
  return new Promise(resolve => {
    inspector.on("boxmodel-view-updated", function onUpdate(reasons) {
      // Wait for another update event if we are waiting for a selection related event.
      if (waitForSelectionUpdate && !reasons.includes("new-selection")) {
        return;
      }

      inspector.off("boxmodel-view-updated", onUpdate);
      resolve();
    });
  });
}
```
We want to listen only "once" to an event, returning a `promise`.
It seems a job for `once` method; it does exactly that.
Unfortunately, we want to do so only *if* certain conditions are met.

Now, the same code can be written has:
```js
function waitForUpdate(inspector, waitForSelectionUpdate) {
  return inspector.once("boxmodel-view-update", {
    when: reasons => !waitForSelectionUpdate || reasons.includes("new-selection")
  });
}
```

2. `on` and `once` can now accept an object as listener à la [DOM EventListener](https://developer.mozilla.org/en-US/docs/Web/API/EventListener)

This was done to avoid to `bind` functions when we're dealing with multiple objects / emitter (kind of actual code):

```js
const EventEmitter = require("devtools/shared/event-emitter");

function Inspector(toolbox) {
  EventEmitter.decorate(this);

  this.onSidebarShown = this.onSidebarShown.bind(this);
  this.onSidebarHidden = this.onSidebarHidden.bind(this);
  this.onSidebarSelect = this.onSidebarSelect.bind(this);

  // Without the `bind`, the contextual object will be `sidebar`
  this.sidebar.on("show", this.onSidebarShown);
  this.sidebar.on("hide", this.onSidebarHidden);
  this.sidebar.on("destroy", this.onSidebarHidden);
 }
```

In regular Web Development it's possible avoid that passing an object, instead of a function, with an [handlerEvent](https://developer.mozilla.org/en-US/docs/Web/API/EventListener) method. I applied the same logic here:

```js
const EventEmitter = require("devtools/shared/event-emitter");

class Inspector extends EventEmitter {
  constructor(toolbox) {
    super();

    this.sidebar.on("show", this);
    this.sidebar.on("hide", this);
    this.sidebar.on("destroy", this);
  }
  
  [EventEmitter.handler](type) {
    switch (type) {
      case "show":
        this.onSidebarShown();
        break;
      case "hide":
      case "destroy":
        this.onSidebarHidden();
        break;
    }
  }
}
```

That avoid to create new functions and rewrite the instance's method all the time, plus it give to us the "master switch" capability if needed.

### Replacing sdk/event/core

Everything should works just replacing the module's path:

Replace:
```js
  const { on, off, emit } = require("sdk/event/core");
```

With:
```js
  const { on, off, emit } = require("devtools/shared/event-emitter");
```

> **Important:** subclassing `EventEmitter` should be always the first attempt, unless the code makes that hard to implement or only `emit` is required. See [Good Practices](#good-practices).

### Replacing sdk/event/target

Use `EventEmitter` instead. There are few differences to keep in mind:

1. `EventTarget`'s methods are chainable; `EventEmitter` aren't (`EventEmitter`'s `once` returns a promise)
2. `EventTarget` has `removeListener(type, listener)`; `EventEmitter` doesn't (use `off` instead)
3. `EventTarget` automatically adds listeners defined in the class; `EventEmitter` doesn't:

    Replace:
    ```js
    const MyCoolComponent = Class({
      extends: EventTarget,
      type: "cool-component",
      initialize: function initialize(name) {
        // this call will automatically bind any `on*` method
        // as listener of this object
        EventTarget.prototype.initialize.call(this);
      },
     onClick: function onClick(data) {
        console.log("clicked! ", data);
      }
    });
    ```

    With:
    ```js
    class MyCoolComponent extends EventEmitter {
      constructor() {
        this.type = "cool-component";
        this.on("click", this.onClick);
      }

      onClick(data) {
        console.log("clicked! ", data);
      }
    }
    ```

### Good practices

#### Prefer ES6 class' `extends` over `EventEmitter.decorate`.

Since `decorate` basically is a mixin and JS doesn't have multiple inheritance, it might be not suitable refactor all the code to extends `EventEmitter`, but it should be our first attempt.

Replace:
```js
const MyCoolComponent = function() {};
MyCoolComponent.prototype = {
  /* ... */
};

EventEmitter.decorate(MyCoolComponent);
```

With:
```js
class MyCoolComponent extends EventEmitter {
  constructor() {
    super();
  }
}
```

### Prefer ES6 class' `extends` over `EventEmitter`'s static methods `on`, `once`

Unless there is a good reason (from API point of view, or code), the `EventEmitter.on` and `EventEmitter.once` shouldn't be used on  generic object.

Replace:
```js
const { on, once } = require("devtools/shared/event-emitter");
const eventBus = {};

on(eventBus, "some-event", doSomething);
once(eventBus, "one-time-only", doSomethingElse);
```

With:
```js
const EventEmitter = require("devtools/shared/event-emitter");
const eventBus = new EventEmitter();

eventBus.on("some-event", doSomething);
eventBus.once("one-time-only", doSomethingElse);
```

That's because internally a [Symbol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol) for the listener is attached to the object, changing its internal Shape; where, instead, the `EventEmitter` object already has it from the start.
That also means, all the `EventEmitter`'s static methods that consumes the listeners instead of adding them, are safe to use (`off`, `emit`, `count`)

### Prefer explicit listeners to a "master switch"

With the experimental functionality that provides a `EventEmitter.handler`, it's possible use a "master switch" as in the example. However, that should be done **only** if we're working with multiple objects, and we need a different contextual object in our listeners. In all the other cases, it's preferable assign any single listener to a specific event type.

### Functionalities not supported

#### sdk/event/utils

To keep it simple, we didn't migrate any of the [event's utils](http://searchfox.org/mozilla-central/source/addon-sdk/source/lib/sdk/event/utils.js) implemented by the SDK. Currently the DevTools codebase shouldn't have any of those, but just as reminder if you have used to the SDK event paradigm before.
