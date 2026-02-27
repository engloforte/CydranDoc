# II. Quick Start

## 3. Getting Started

### Basic Setup (Single Component Only)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Cydran Single Component</title>
</head>
<body>
  <div id="app"></div>

  <template id="appTemplate">
    <main>
      <h1>{{m().title}}</h1>
      <p>This page renders one Cydran component.</p>
    </main>
  </template>

  <script src="./cydran.js"></script>
  <script>
    const APP_TEMPLATE =
      document.querySelector('template[id="appTemplate"]')?.innerHTML.trim() ?? '';

    class App extends cydran.Component {
      constructor() {
        super(APP_TEMPLATE);
        this.title = 'Hello from a single component';
      }
    }

    function rootCapability(context) {
      context.registerSingleton('app', App);
    }

    const root = cydran.create('#app', {}, rootCapability);
    root.addInitializer(null, s => {
      s.setComponentByObjectId('app');
    });
    root.start();
  </script>
</body>
</html>
```

### Parent-to-Child Data with `<c-region value="...">`

Use `value` on `<c-region>` when the parent needs to pass context data to the child.

```html
<template id="appTemplate">
  <main>
    <h1>{{m().title}}</h1>
    <c-region name="detailsRegion" path="detailsPanel" value="m().selectedItem"></c-region>
  </main>
</template>
```

```javascript
class DetailsPanel extends cydran.Component {
  constructor() {
    super('<section>{{m().label}}</section>');
    this.label = '';
  }

  onMount() {
    const item = this.$c().getValue(); // value from parent c-region
    this.label = item?.label ?? 'No item selected';
  }
}

function rootCapability(context) {
  context.registerSingleton('app', App);
  context.registerPrototype('detailsPanel', DetailsPanel);
}
```

Notes:
- Use `value`, not `c-value`.
- `value` sets child context (`$c().getValue()`); constructor args still come from `getObject(...args)` + `withArgument(index)`.

### The Root Component

The **root component** is the root container for your application:
- Created with `cydran.create(selector, properties, capabilityFunction)`

- Binds the component tree to an element in the DOM (ex. Attach App Component to the #app element in `index.html`)
- Handles initialization and lifecycle

Arguments in `create(selector, properties, capabilityFunction)`:
- `selector`: CSS selector for the host DOM element (for example `'#app'`).
- `properties`: configuration object for Cydran/runtime settings (logging, strict mode, and custom properties). Use `{}` when you do not need custom settings.
- `capabilityFunction`: setup callback that receives the root context so you can register and configure everything the app needs (`registerSingleton`, `registerPrototype`, scope functions, properties-driven wiring, etc.).

Execution timing for `capabilityFunction`:
- It runs during root creation/setup, before the app is activated with `root.start()`.
- No component is rendered/mounted yet when this callback executes.
- Use this phase to guarantee registrations are in place before any initializer or component resolution runs.
- In practice: treat it as your composition/bootstrap step so all pieces are configured before rendering begins.

---

## 4. The Root Component

The **root component** is the root container for your application:
- Created with `cydran.create(selector, properties, capabilityFunction)`

- Binds the component tree to an element in the DOM (ex. Attach App Component to the #app element in `index.html`)
- Handles initialization and lifecycle

Arguments in `create(selector, properties, capabilityFunction)`:
- `selector`: CSS selector for the host DOM element (for example `'#app'`).
- `properties`: configuration object for Cydran/runtime settings (logging, strict mode, and custom properties). Use `{}` when you do not need custom settings.
- `capabilityFunction`: setup callback that receives the root context so you can register and configure everything the app needs (`registerSingleton`, `registerPrototype`, scope functions, properties-driven wiring, etc.).

Execution timing for `capabilityFunction`:
- It runs during root creation/setup, before the app is activated with `root.start()`.
- No component is rendered/mounted yet when this callback executes.
- Use this phase to guarantee registrations are in place before any initializer or component resolution runs.
- In practice: treat it as your composition/bootstrap step so all pieces are configured before rendering begins.

---

## 5. Basic Project Checklist

Use this list to bootstrap a new project quickly:

1. **Create HTML shell**
   - Add a root element (e.g., `<div id="app"></div>`).
   - Add `<template>` tags for each component.
   - Include `cydran.js` and your app script(s).

2. **Define a template loader**
   - Implement a `template(id)` helper that returns HTML strings.
   - Map component names to template ids.

3. **Create the root component**
   - Extend `cydran.Component`.
   - Initialize state in `constructor()`.
   - Use `onMount()` for DOM- or data-dependent work.

4. **Register components and services**
   - `registerSingleton` for shared services or the root component.
   - `registerPrototype` for repeated components.

5. **Start the root**
   - `const root = cydran.create('#app', PROPERTIES, rootCapability);`
   - `root.addInitializer(null, s => s.setComponentByObjectId('app'));`
   - `root.start();`

6. **Add communication and placement**
   - Use `$c().onMessage(...)` for messaging.
   - Use `c-region` + `regions().set(...)` for child placement.
   - Use `c-series` + `forSeries(...)` for lists of components.

7. **Enable helpful settings (optional)**
   - Strict mode and logging via `PropertyKeys`.

This checklist plus the API reference should be enough to start a new Cydran project without retraining.

---

