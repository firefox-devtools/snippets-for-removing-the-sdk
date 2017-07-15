# snippets-for-removing-the-sdk
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
const { extend } = require("devtools/shared/DevToolsUtils");
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
