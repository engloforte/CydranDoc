# Cydran Documentation

## Table of Contents
1. [Overview](#overview)
2. [Core Concepts](#core-concepts)
3. [Getting Started](#getting-started)
4. [Components](#components)
5. [Templates](#templates)
6. [Data Binding](#data-binding)
7. [Messaging & Communication](#messaging--communication)
8. [Dependency Injection](#dependency-injection)
9. [Lifecycle Hooks](#lifecycle-hooks)
10. [Advanced Features](#advanced-features)
11. [Best Practices](#best-practices)

---

## Overview

**Cydran** is an unobtrusive JavaScript/TypeScript MVVM (Model-View-ViewModel) presentation framework designed for building interactive web applications. This project uses a simple Hello World component to demonstrate:

- **Declarative UI**: HTML templates with data binding
- **Component-Based Architecture**: Reusable, self-contained components
- **MVVM Pattern**: Clear separation of concerns
- **Message-Driven Communication**: Decoupled component interactions
- **Dependency Injection**: Built-in IoC container

The framework is lightweight and can be used in applications ranging from simple to complex.

### Foundational Programming Practices

Cydran is built on a set of core engineering practices:

1. Separation of concerns (MVVM + component boundaries):
   - Template, model, and framework orchestration are deliberately separated.

2. Declarative UI:
   - Templates and bindings drive rendering/state synchronization instead of manual DOM wiring.

3. Dependency injection and composition root:
   - App wiring is centralized in `rootCapability(context)` with explicit registrations.

4. Message-driven decoupling:
   - Components/services communicate through channels and propagation scopes (Pub/Sub) rather than direct references.

5. Explicit lifecycle/resource management:
   - `constructor`, `onMount`, `onUnmount`, `onRemount`, `$release` define controlled setup/teardown points.

6. State modeling with transitions:
   - State machines provide explicit transition/guard behavior for workflow-heavy features.

7. Scoped modularity:
   - Contexts constrain dependency resolution and message routing for feature isolation.

8. Contract-driven APIs:
   - Constants, enums, and typed API surfaces reduce magic strings and improve maintainability.

9. Observability:
   - Logging and diagnostics are first-class and configurable through properties.

10. Practical best-practice orientation:
   - The framework encourages clear module boundaries, small payload events, and lifecycle-safe code.

---

## Core Concepts

### MVVM Pattern

Cydran follows the Model-View-ViewModel pattern:

- **Model**: The component with data and display functionality
- **View**: The HTML templates with binding expressions
- **ViewModel**: The layer (cydran core) that keeps the View and Model in sync

### Components

Components are the building blocks of Cydran applications and encapsulate:
- A template (HTML) - Developer
- A model (JavaScript/TypeScript class) - Developer
- Event handling and data binding - Cydran
- Lifecycle management - Cydran

### Non-Cydran Components

Not everything in a Cydran app should be a UI component. Plain objects can participate through the IoC context without extending `cydran.Component`.

- **Services**: business logic and orchestration.
- **Repositories**: data access and persistence.
- **Utilities/Adapters**: API clients, formatters, browser/storage wrappers.

How they participate:
- Register them in `rootCapability(context)` as singleton/prototype.
- Resolve them from components with `$c().getObject(...)`.
- Keep UI state in components; keep domain/data logic in these objects.

### Two-Way Data Binding

Changes to model properties automatically update the view, and user interactions update the model.

---

## Getting Started

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

## Components

### Creating a Component

To create a component, you need:

- A template source (string, DOM element, or renderer) passed to `super(...)`.
- A constructor that calls `super(...)` first.
- Component state fields on `this` for values your template binds to.
- Optional constructor dependencies, if you register the component with argument resolvers.

Basic shape:

```javascript
const template = '<div>{{m().greeting}}</div>';

class HelloWorld extends cydran.Component {
  constructor() {
    super(template); // required first call
    this.greeting = 'Hello World!';
  }
}
```

Using `super(...)`:
- `super(template)` for the common case.
- `super(template, options)` when you need `ComponentOptions` (name/prefix/styles/metadata).

Constructor properties/state:
- Define fields that represent UI state (`this.greeting`, `this.items`, `this.loading`).
- Keep constructor work focused on initial state and wiring.
- Put DOM/data-loading work in `onMount()` when it depends on mounted state or services.

Constructor dependencies (optional):
- Register the component with resolvers (for example `withObject(...)` or `withArgument(...)`).
- Accept those injected values as constructor parameters and pass template/options through `super(...)`.

Example with dependency:
```javascript
class UserPanel extends cydran.Component {
  constructor(userService) {
    super('<section>{{m().name}}</section>');
    this.userService = userService;
    this.name = 'Loading...';
  }

  onMount() {
    this.name = this.userService.currentUserName();
  }
}
```

### ComponentOptions

`ComponentOptions` can be passed as the second argument to `super(template, options)` (or to `registerImplicit(...)`). It lets you customize how the component is named, scoped, and styled.

Fields:
- `name` (string): Override the component name.
- `prefix` (string): Override the binding prefix used by the template extractor.
- `styles` (string): Component-scoped CSS injected via `<c-component-styles>`.
- `metadata` (SimpleMap): Initial metadata to attach to the component.

Example:
```javascript
const template = '<div class="card"><c-component-styles></c-component-styles>{{m().title}}</div>';

class Card extends cydran.Component {
  constructor() {
    super(template, {
      name: 'Card',
      styles: '.card { border: 1px solid #c9bba8; padding: 8px 12px; }'
    });
    this.title = 'Card title';
  }
}
```

### Component Context ($c()) Reference

Every component has access to a context object via `this.$c()`. The API is grouped below by purpose.

#### Identity and Hierarchy

| Method | Purpose |
|--------|---------|
| `getId()` | Unique runtime id for this component instance |
| `getName()` | Component name (often class name) |
| `getParent()` | Parent component (or `null` at root) |
| `isConnected()` | Whether the component is connected in the component tree |
| `isMounted()` | Whether the component is currently mounted in the DOM |

#### Context and Dependency Access

| Method | Purpose |
|--------|---------|
| `getContext()` | Access the component IoC context |
| `getObject(name, ...args)` | Resolve an object from IoC; use `...args` for runtime inputs mapped by `withArgument(index)` |
| `properties()` | Access runtime properties (`PropertyKeys` map) |
| `getLogger()` | Get the component-scoped logger |
| `metadata()` | Access metadata helpers (`get`, `has`) |

#### Model, Scope, and Evaluation

| Method | Purpose |
|--------|---------|
| `getValue()` | Get this component's current model/value |
| `scope()` | Access the template scope |
| `getPrefix()` | Get the binding prefix used by template extraction |
| `getWatchScope()` | Access watch scope for expression evaluation |
| `evaluate(expression)` | Evaluate an expression in component scope |
| `sync()` | Manually trigger a digestion cycle when state changes happen outside normal Cydran-driven events |

Use `sync()` when data changes occur asynchronously or outside Cydran event bindings (for example API callbacks, timers, or third-party library events). After updating component fields, call `this.$c().sync()` so Cydran digests changes and refreshes bound template values.
For standard template event handlers (`c-on*`), Cydran typically runs digest/sync automatically at the end of the handler, so manual `sync()` is usually not needed there.

#### Reactivity and Filters

| Method | Purpose |
|--------|---------|
| `onExpressionValueChange(expr, callback, [thisArg], [mutable])` | Watch expression changes and run callback |
| `onInterval(delayMs)` | Run a callback on a repeating interval |
| `createFilter(expression)` | Build a reactive filtered collection |

#### Messaging

| Method | Purpose |
|--------|---------|
| `onMessage(messageName)` | Subscribe to messages (chain `.forChannel(...)`) |
| `send(messageName, [payload], [startFrom])` | Send a message (chain `.onChannel(...)`) |
| `tell(name, payload)` | Send an internal framework message |

#### DOM, Regions, Series, and Forms

| Method | Purpose |
|--------|---------|
| `getEl()` | Get the root DOM element for this component |
| `regions()` | Access region APIs (set/get/has) |
| `forSeries(name)` | Access a `c-series` by name |
| `forElement(name)` | Access a component-local named element registered via `c-id` (no global `id` required) |
| `forForm(name)` | Access a named form registered via `c-id` |
| `forForms()` | Access all forms registered on the component |

`forElement(name)` is used to tag elements inside a component template for later access from that component. This solves global `id` collision problems: if multiple component instances render the same markup, repeated global ids like `id="saveButton"` would clash in the document. Using `c-id` + `forElement(...)` keeps lookup scoped to the component instance, so each instance can safely have its own `"saveButton"` element name.

`forForm(name)` returns one named form registered with `c-id`. `forForms()` returns the full component-local form registry, which is useful when you need to run the same operation across every registered form (for example, validating or resetting all forms in the component).

### Regions() API

```
regions().has(name)
regions().get(name)
regions().set(name, componentOrNull)
regions().setByObjectId(name, objectId, [fallbackObjectId])
```

---


## Markers & Placeholders

Cydran uses a few structural markers/placeholders in the DOM. These are normal and help the framework manage dynamic insertion/removal.

- **`<c-region>`**: A placeholder tag that is replaced by the mounted component for that region.
- **`<c-series>`**: A placeholder tag that is replaced by a series insertion point.

### Tip for Debugging: `data-template-id`

Adding `data-template-id="..."`can aid DOM inspection.

## Custom Elements

Cydran registers the following custom elements. These are real custom tags (via `customElements.define`) and are parsed by the framework during template compilation.

### `<c-region>`

Region placeholder used to mount child components.

Use `value` for parent-to-child data context. There is no `c-value` attribute for `<c-region>`.

| Attribute | Type | Purpose | Notes |
|----------|------|---------|-------|
| `name` | string | Names the region so it can be targeted from code | If omitted, Cydran generates an anonymous region name |
| `path` | string | Requestable object path to a component in the IoC context | If set, the region auto-loads that component on mount |
| `value` | expression | Expression providing item/value context to the child component | Used as the region's item function (`setItemFn`) |
| `lock` | boolean attribute | Locks the region from further updates | If `true`, updates throw `LockedRegionError` |

Notes:
- If `path` is set and `name` is not, the region is locked by default (cannot be changed later).
- The `value` expression is evaluated in the parent scope and assigned as the region item function (`setItemFn`).
- The `value` expression is re-evaluated when the region/child is (re)mounted, so child context reflects the current parent expression value at mount time.
- Child components placed in the region can read that context value via `$c().getValue()`.
- `value` context and `getObject(name, ...args)` are different inputs:
  - `value` sets child context/value (`$c().getValue()`).
  - `...args` map to constructor inputs via `withArgument(index)`.

Example:
```html
<c-region name="detailsRegion" path="detailsPanel" lock></c-region>
```

Lock behavior example:

```javascript
// If the region is locked, this update throws LockedRegionError
this.$c().regions().set("detailsRegion", null);
```

Parent/child value context example:

```html
<!-- parent template -->
<c-region
  name="detailsRegion"
  path="detailsPanel"
  value="m().selectedItem">
</c-region>
```

```javascript
// child component
onMount() {
  const selectedItem = this.$c().getValue();
  // use selectedItem
}
```

Framework source references:
- `RegionTagBuilder` reads `value`: `src/template/regions/RegionTagBuilder.ts`
- `RegionLinkerImpl` applies it with `setItemFn`: `src/template/regions/RegionLinkerImpl.ts`

Troubleshooting:
- If the child sees `undefined` from `$c().getValue()`, verify:
  - You used `value="..."` on `<c-region>` (not `c-value`).
  - The expression resolves in the parent scope.
  - The child is mounted in that region (and not replaced/cleared immediately).

### `<c-series>`

Series placeholder used to insert and manage ordered child components.

| Attribute | Type | Purpose | Notes |
|----------|------|---------|-------|
| `name` | string | Names the series so it can be targeted from code | Use with `this.$c().forSeries(name)` |

Example:
```html
<c-series name="nodes"></c-series>
```

```javascript
const series = this.$c().forSeries("nodes");
series.insertLast(this.$c().getObject("nodeItem", [node]));
```

### `<c-component-styles>`

Placeholder for component-scoped CSS. During compilation, this element is replaced by a `<style>` tag that contains the component's styles (if any). See **ComponentOptions** for how to supply the `styles` string.

| Attribute | Type | Purpose | Notes |
|----------|------|---------|-------|
| *(none)* | - | - | This tag is replaced with a `<style>` element |

Example:
```html
<c-component-styles></c-component-styles>
```

**Binding a component to styles**

`<c-component-styles>` is a placeholder that Cydran replaces with a `<style>` tag for the **current component**. The style text is sourced from the componentâ€™s registered template/styles so the rules apply only within that componentâ€™s scope.

Example (component HTML includes the placeholder, CSS is scoped to this component):
```html
<div class="card">
  <c-component-styles></c-component-styles>
  <h3>{{m().title}}</h3>
</div>
```

```css
.card {
  border: 1px solid #c9bba8;
  padding: 8px 12px;
  border-radius: 6px;
}
```

If the component has no registered styles, the placeholder is removed without adding a `<style>` tag.

## Templates

### Template Definition

Templates are defined as HTML strings inside `<template>` elements:

```html
<template id="hello-world">
  <div class="container">
    <h1>{{m().greeting}}</h1>
    <p>{{m().message}}</p>
    <button c-onclick="m().updateGreeting()">Click Me!</button>
    <button c-onclick="m().showMessage()" c-enabled="!m().messageShown">Show Message</button>
    <p>Button clicked: {{m().clickCount}} times</p>
    <c-region name="messageRegion"></c-region>
  </div>
</template>

<template id="simple-message">
  <div class="simple-message">
    <p>{{m().notice}}</p>
    <button c-onclick="m().hideMe()">Hide Message</button>
  </div>
</template>
```

### One Top-Level Node Required

Each template must have exactly **one** root element. Text nodes (including stray whitespace) and comment nodes at the top level are not permitted. If you need multiple elements, wrap them in a single container.

Valid:
```html
<template id="example">
  <div class="wrapper">
    <h1>{{m().title}}</h1>
    <p>{{m().body}}</p>
  </div>
</template>
```

Invalid (multiple top-level nodes):
```html
<template id="example">
  <h1>{{m().title}}</h1>
  <p>{{m().body}}</p>
</template>
```

### What `super(...)` Accepts

`super(...)` can take:
- A **template string**
- An **HTMLElement** (pre-built DOM to use as the root)
- A **renderer object** with a `render()` function

**Renderer objects** let you generate the DOM programmatically. The object must provide a `render()` method that returns a **single root `HTMLElement`** for the component. The return value is used as the componentâ€™s root element, so it must be:
- An actual `HTMLElement` (not a string, not a `DocumentFragment`).
- A **single root** (no sibling root nodes).
- Fully constructed before returning (children appended inside the root).

### SVG Fragment Templates

When your template is an **SVG fragment** (for example, a `<g>` block), a renderer helps ensure the fragment is parsed in the SVG namespace before Cydran binds it. This avoids browsers treating the markup as plain HTML and dropping SVG-specific attributes.

Example:
```javascript
import { svgTemplateRenderer } from "../lib/template";

class LinksLayer extends cydran.Component {
  constructor() {
    super(svgTemplateRenderer('<g class="links-layer"><g class="link-lines"></g></g>'));
  }
}
```

Example:
```javascript
class FancyBox extends cydran.Component {
  constructor() {
    super({
      render() {
        const el = document.createElement('div');
        el.className = 'fancybox';
        el.textContent = 'Rendered by function';
        return el;
      }
    });
  }
}
```

### Retrieving Templates

Use a helper function to retrieve template HTML:

```javascript
const template = id =>
  document.querySelector(`template[id="${id}"]`).innerHTML.trim();

// Usage in component
super(template('hello-world'));
```

You can also align template ids to component names:

```javascript
const template = id =>
  document.querySelector(`template[id=${id}]`).innerHTML.trim();

class App extends Component {
  constructor() {
    super(template(App.name.toLowerCase()));
  }
}
```

### Template Sources (All Supported Approaches Used in This Project)

You can supply templates in several ways. These are all valid and supported by Cydran, and we have used each in this project:

1) **Inline `<template>` tags in `index.html`**
   - Define templates in the page and load them by id.
   - Works well for small apps or quick prototypes.

```html
<template id="app-root">
  <div class="app-root">...</div>
</template>
```

```javascript
const template = id =>
  document.querySelector(`template[id="${id}"]`).innerHTML.trim();

super(template("app-root"));
```

2) **Separate HTML files imported as raw strings (Vite/esbuild)**
   - Put each component template in its own `.html` file.
   - Import the file as a raw string and pass it directly to `super(...)`.
   - The HTML file can be either:
     - a **fragment** (preferred now), or
     - a `<template>` wrapper (older style) that you strip yourself before passing.

```javascript
import templateHtml from "./app-root.html?raw";

// If the file is a fragment, pass it directly.
super(templateHtml);
```

3) **Central template registration (legacy pattern)**
   - Import raw template files, register them, then load by id in components.
   - Useful when you want a single place to manage templates or keep `super(...)` calls consistent.

```javascript
import appRootTemplate from "./components/app-root/app-root.html?raw";
import { registerTemplates } from "./lib/register-templates";

registerTemplates([appRootTemplate]);

super(template("app-root"));
```

**Notes**
- If you pass a fragment directly to `super(...)`, it should contain **one top-level element**.
- If you use `<template>` wrappers, ensure you extract the inner HTML before passing to `super(...)`.

### Template Syntax

#### Data Binding (`{{ }}`)
Reactive interpolation. The expression is evaluated in the component scope and will update the DOM when its value changes. You can use `m()`, `v()`, `s()`, and `p()` inside the expression.
```html
<h1>{{m().greeting}}</h1>
<p>{{m().message}}</p>
<p>Button clicked: {{m().clickCount}} times</p>
```

#### Alternate Binding (`[[ ]]`)
`[[ ]]` is supported as a **one-time** interpolation. It writes the initial value but does not update on subsequent model changes.
```html
<div aria-valuenow="[[m().value]]"></div>
<span>[[m().label]]</span>
```

#### Property Binding (`c-model`)
Two-way binding to model properties:
```html
<input c-model="m().greeting">
<textarea c-model="m().message"></textarea>
```

#### Event Binding (`c-on*`)
Bind to DOM events:
```html
<button c-onclick="m().updateGreeting()">Click Me!</button>
<button c-onclick="m().showMessage()">Show Message</button>
<button c-onclick="m().hideMe()">Hide Message</button>
```
When the DOM event fires, Cydran evaluates the expression in component scope (same context as `{{ ... }}`), invokes the target function, and then runs a digest to reconcile view updates. The digest is what refreshes bound values after state changes.

#### Conditional Rendering (`c-if`)
Show/hide elements based on conditions:
```html
<section c-if="m().clickCount > 0">
  <!-- Content shown only after a click -->
</section>
```

#### List Rendering (`c-each`)
Render lists of items:
```html
<ul c-each="m().messages" c-each-mode="none">
  <li>{{v()}}</li>
</ul>
```
For `c-each` on non-`<template>` elements, Cydran requires a single child `<template c-type="item">` that defines the repeated markup:
```html
<div c-each="m().items" c-each-mode="none">
  <template c-type="item">
    <div class="row">{{v().label}}</div>
  </template>
</div>
```
If you see an error about a missing id for repeat items, set a key with `c-each-idkey`. The id field must exist on each item and be string-convertible:
```html
<div c-each="m().items" c-each-mode="none" c-each-idkey="id">
  <template c-type="item">
    <div class="row">{{v().label}}</div>
  </template>
</div>
```
For large lists, use `c-each-mode="generated"` with `c-each-idkey` so Cydran can keep stable ids and avoid full rebuilds.

#### Class Binding (`c-class`)
Dynamically apply CSS classes:
```html
<div c-class="{'highlight': m().clickCount > 0}">
```
`c-class` merges with existing class names. To remove a class, include it with a `false` value in the object so it is explicitly cleared.

#### Attribute Binding
```html
<input c-enabled="m().clickCount < 10">
<span c-enabled="m().clickCount < 10"></span>
```

For non-`c-` attributes, use `{{ ... }}` for reactive updates. Use `[[ ... ]]` for **write-once** values (it does not stay reactive):

```html
<div aria-valuenow="{{m().value}}"></div>
<div style="{{ 'width: ' + m().value + '%' }}"></div>

<div aria-valuenow="[[m().value]]"></div>
<div style="[[ 'width: ' + m().value + '%' ]]"></div>
```

#### Focus Management (`c-force-focus`)
```html
<input c-force-focus="m().clickCount === 0">
```

### c-region Modifiers

See **Custom Elements â†’ `<c-region>`** for the full attribute table and examples.

### Series Placement (c-series)
**DOM markers (`<!--SS-->` / `<!--SE-->`)**
When Cydran manages a `c-series`, it may insert HTML comment markers like `<!--SS-->` and `<!--SE-->` in the DOM. These mark the start/end boundaries for the series so the framework can insert, remove, and reorder components without needing a wrapping element. Seeing these comments inside a `c-series` container is normal.
**Using getObject with series items**
When inserting components into a series, use `getObject(name, ...args)` to pass instance arguments to prototypes registered with `withArgument(index)`. The `withArgument(0)` value comes from the first element of the `getObject(...instanceArguments)` list (argProperties).

Example:
```javascript
const series = this.$c().forSeries('nodes');
const comp = this.$c().getObject('nodeItem', enrichedNode);
series.insertLast(comp);
```

For multiple child components, use `c-series` plus `forSeries(...)`.

Template:
```html
<c-series name="items"></c-series>
```

Code placement APIs:
```javascript
const series = this.$c().forSeries('items');
series.insertFirst(component);
series.insertLast(component);
series.insertBefore(index, component);
series.insertAfter(index, component);
series.replace(oldComponent, newComponent);
series.replaceAt(index, component);
series.remove(component);
series.removeAt(index);
series.clear();
```

Notes:
- Series prevent duplicate components.
- `getAt(index)`, `contains(component)`, `hasComponents()`, and `isEmpty()` help inspect series state.
- Performance: `insertLast(...)` is the fastest insertion path; `insertFirst(...)` is fast when empty but otherwise delegates to `insertBefore(0, ...)`. `insertBefore/insertAfter` do extra bounds checks and array splices.

### Template Context Functions

Within templates, you can use:

| Function | Purpose | Example |
|----------|---------|---------|
| `m()` | Access view model | `m().greeting` |
| `v()` | Access component value (in iterators) | `v().title` |
| `s()` | Access scope functions | `s().pluralize('item', 3)` |
| `p()` | Access parameters (in event handlers) | `p().$event` |
| `c-region` | Named region placeholder | `<c-region name="messageRegion"></c-region>` |

---

## Data Binding

### Two-Way Binding

```html
<input c-model="m().title">
```

When the user types, the model updates. When the model updates programmatically, the view updates.

### Watched Expressions

Watch for changes to specific expressions:

```javascript
this.$c().onExpressionValueChange('m().clickCount', () => {
  // React to click count changes
});
```

You can also watch values inside iterators:

```javascript
this.$c().onExpressionValueChange('v().completed', () => {
  this.sendUpdate();
});
```

### Filtered Collections

Create filtered arrays with predicates:

```javascript
this.filtered = this.$c().createFilter('m().messages')
  .withPredicate("v().includes('click')", 'm().clickCount')
  .build();
```

In the template:
```html
<ul c-each="m().filtered.items()">
  <li>{{v()}}</li>
</ul>
```

---

## Messaging & Communication

### Message-Based Architecture

Components communicate through **channels** and **messages**, enabling loose coupling:

```javascript
// Define channel and message constants
const APP_CHANNEL = 'APP';
const CLICKED = 'Clicked';
const HIDE_MESSAGE = 'hideMessage';
```

### Sending Messages

Components send messages using the transmitter:

```javascript
// In a component
notifyClick() {
  this.txmitr.send(To.GLOBALLY, APP_CHANNEL, CLICKED, this.clickCount);
}
```

Send from a child component to its parent:

```javascript
this.$c()
  .send(HIDE_MESSAGE)
  .onChannel(APP_CHANNEL)
  .withPropagation(cydran.To.PARENT);
```

Send parameters:
- `To.GLOBALLY`: Send to all components
- `APP_CHANNEL`: Channel name
- `CLICKED`: Message name
- Data payload (can be any object)

### Scope Examples

```javascript
const channel = APP_CHANNEL;
const message = "REFRESH";
const payload = { source: "example" };

this.$c().send(message, payload).onChannel(channel).withPropagation(cydran.To.GLOBALLY);
this.$c().send(message, payload).onChannel(channel).withPropagation(cydran.To.CONTEXT);
this.$c().send(message, payload).onChannel(channel).withPropagation(cydran.To.DESCENDANTS);
this.$c().send(message, payload).onChannel(channel).withPropagation(cydran.To.IMMEDIATE_CHILDREN);
this.$c().send(message, payload).onChannel(channel).withPropagation(cydran.To.IMMEDIATE_CHILD_COMPONENTS);
this.$c().send(message, payload).onChannel(channel).withPropagation(cydran.To.PARENT);
this.$c().send(message, payload).onChannel(channel).withPropagation(cydran.To.PARENTS);
this.$c().send(message, payload).onChannel(channel).withPropagation(cydran.To.ROOT);
```

### Contexts

A Cydran context is a scoped runtime boundary for dependency resolution and message routing.

- DI scope: objects registered in a context are resolved relative to that context boundary.
- Messaging scope: `To.CONTEXT` keeps delivery inside the current context.
- Isolation: feature modules can stay local and avoid app-wide coupling.

DI example (context-local registrations):

```javascript
function rootCapability(context) {
  context.registerSingleton("todoRepo", TodoRepo);
  context.registerPrototype("todoItem", TodoItem, cydran.argumentsBuilder().with("todoRepo").build());
}
```

Messaging example (stay in local context):

```javascript
this.$c()
  .send("ITEM_SELECTED", { id })
  .onChannel("todo")
  .withPropagation(cydran.To.CONTEXT);
```

### Receiving Messages

Subscribe to messages in the constructor:

```javascript
this.$c().onMessage(CLICKED)
  .forChannel(APP_CHANNEL)
  .invoke(count => {
    this.clickCount = count;
  });

this.$c().onMessage(HIDE_MESSAGE)
  .forChannel(APP_CHANNEL)
  .invoke(() => {
    this.hideMessage();
  });
```

**Non-component receivers (services/brokers)**

Non-component classes are not auto-subscribed to the messaging broker. If you inject a `Receiver` into a service, you must also inject a message subscriber and register the receiver:

```javascript
// registration
context.registerSingleton(
  'apiBroker',
  ApiBroker,
  argumentsBuilder().with('apiService').withReceiver().withTransmitter().withMessageSubscriber().build()
);

// in ApiBroker constructor
constructor(api, rx, tx, subscribe) {
  this.api = api;
  this.rx = rx;
  this.tx = tx;
  subscribe(this.rx, this.rx.message);
}

// outbound publish from non-component class
publishUpdate(payload) {
  this.tx.send(cydran.To.GLOBALLY, "API_CHANNEL", "API_UPDATED", payload);
}
```

What `subscribe(...)` does:
- Registers the instance receiver (`rx`) with Cydran's message broker.
- Binds `rx.message` as the callback the broker invokes for delivered messages.
- Activates the instance as a Pub/Sub endpoint.
- Without this registration, the instance can publish (if it has a transmitter) but will not receive broker-delivered messages.

**`invoke` and `this`**

`invoke(...)` calls your handler with the component instance as `this`. Passing a method reference (e.g. `this.removeTodo`) is fine because Cydran binds `this` to the component when invoking. Arrow functions still work, but they ignore rebinding and capture `this` lexically instead.

### Debugging Messaging

When debugging message flow, log before sending and inside the receive handler. Include the message name, channel, and propagation:

```javascript
// Send
console.log('db:SEND', TREE_UPDATED, APP_CHANNEL, cydran.To.DESCENDANTS);
this.$c()
  .send(TREE_UPDATED, payload)
  .onChannel(APP_CHANNEL)
  .withPropagation(cydran.To.DESCENDANTS);

// Receive
this.$c()
  .onMessage(TREE_UPDATED)
  .forChannel(APP_CHANNEL)
  .invoke(() => {
    console.log('db:RECV', TREE_UPDATED, APP_CHANNEL, 'propagation=n/a');
    // handler code...
  });
```

Tips:
- Use consistent prefixes (`db:SEND`, `db:RECV`) so you can filter in the console.
- Log payload sizes or keys rather than full objects when payloads are large.
- If a message seems "lost", verify the channel and propagation target match the sender.

### Message-Driven Patterns

Messages enable:
- Parent-child communication
- Sibling component communication
- Global event broadcasting
- Decoupled architecture

### Pub/Sub Best Practices

- Use clear message contracts:
  - Define message constants and payload schema in one place.
  - Prefer small payloads (ids + changed fields), not full mutable objects.
  - Add optional `messageId`, `correlationId`, and `source` fields for tracing.
- Choose the narrowest propagation scope that works:
  - Prefer `PARENT`/`CONTEXT` before `GLOBALLY`.
  - Keep channels feature-scoped (for example `todo:*`, `tree:*`).
- Keep handlers idempotent and loop-safe:
  - Handlers should be safe if called more than once.
  - Avoid re-publishing the same event from its own handler unless you gate it.
- Manage lifecycle and subscriptions:
  - Register component subscriptions once (typically constructor).
  - For non-component receivers, wire `subscribe(...)` exactly once per instance.
  - Clean up any manual external subscriptions/listeners in `onUnmount()`/`$release()`.
- Handle errors explicitly:
  - Wrap risky handler logic in `try/catch`.
  - Log message name/channel/scope plus correlation metadata.
  - Fail fast on invalid payloads (missing required keys).
- Use Pub/Sub for events, not direct request/response:
  - Use DI/service calls for synchronous queries or command return values.

---

## Dependency Injection

### Registering Components

In the root capability function, register components with the IoC container:

```javascript
function rootCapability(context) {
  // Register a singleton (one instance for the app)
  context.registerSingleton('app', HelloWorld);
}
```

**Singleton vs Prototype**

- **Singleton**: One shared instance for the entire app. Use for services, repositories, and root-level components that should be unique (e.g., `TodoRepo`, the root `App` component).
- **Prototype**: A new instance is created each time the container is asked for one. Use for components that appear multiple times (e.g., list items like `TodoItem`, dialog instances, or routed views).

Register a prototype that receives a transmitter:

```javascript
context.registerPrototype(TodoItem.name, TodoItem,
  ab().withTransmitter().build()
);
```

### Registering an Entire Module

When a feature has many services/components, group registrations into a module installer so the root registers the whole module with one call.

```javascript
const ab = cydran.argumentsBuilder;

const InventoryModule = {
  services: [
    { id: "inventoryApi", type: "singleton", cls: InventoryApi },
    { id: "inventoryRepo", type: "singleton", cls: InventoryRepo, args: () => ab().with("inventoryApi").build() }
  ],
  components: [
    { id: "inventoryApp", type: "singleton", cls: InventoryApp },
    { id: "inventoryList", type: "prototype", cls: InventoryList, args: () => ab().with("inventoryRepo").build() },
    { id: "inventoryDetails", type: "prototype", cls: InventoryDetails, args: () => ab().with("inventoryRepo").build() }
  ],
  entry: "inventoryApp"
};

function registerModule(context, moduleDef) {
  const all = [...moduleDef.services, ...moduleDef.components];
  for (const item of all) {
    const resolvers = item.args ? item.args() : undefined;
    if (item.type === "singleton") {
      context.registerSingleton(item.id, item.cls, resolvers);
    } else {
      context.registerPrototype(item.id, item.cls, resolvers);
    }
  }
}

function rootCapability(context) {
  registerModule(context, InventoryModule); // single registration call
}

const root = cydran.create("#app", {}, rootCapability);
root.addInitializer(null, s => s.setComponentByObjectId(InventoryModule.entry));
root.start();
```

Module checklist:
- Register services/repositories before dependent components.
- Use one module entry component (`entry`) for startup.
- Namespace module message channels/constants (for example `inventory:*`).

### ArgumentsBuilder

Use `argumentsBuilder` (aliased as `ab`) to configure component dependencies:

```javascript
const ab = cydran.argumentsBuilder;

// Inject logger and transmitter
ab()
  .withLogger('MyComponent')
  .withTransmitter()
  .build()
```

#### Factory Registrations (`register*WithFactory`)

Use factory registrations when object construction needs custom runtime logic beyond straightforward constructor wiring.

```javascript
function rootCapability(context) {
  // Singleton factory: created once, shared everywhere
  context.registerSingletonWithFactory("clock", () => new ClockService(Date.now()));

  // Prototype factory: new instance each resolve
  context.registerPrototypeWithFactory("todoItemVm", () => {
    return new TodoItemViewModel();
  });
}
```

When to use factory registration:
- Construction depends on conditional logic.
- You need to compose/decorate an object before exposing it.
- Constructor arguments do not map cleanly to `ArgumentsBuilder`.

When class registration is better:
- Constructor dependencies are stable and declarative.
- You want argument mapping documented via `withConstant(...)` / `withArgument(...)`.

#### Passing Constructor Arguments (withConstant / withArgument)

Use `withConstant(...)` to pass literal values into the constructor. Use `withArgument(index)` **only** when you plan to pass instance arguments at resolve-time (the index pulls from the instance-args array).

```javascript
const ab = cydran.argumentsBuilder;

function rootCapability(context) {
  context.registerPrototype('widgetHorizontal', Widget,
    ab().withConstant(TEMPLATE_HORIZONTAL).build()
  );

  context.registerPrototype('widgetVertical', Widget,
    ab().withConstant(TEMPLATE_VERTICAL).build()
  );
}
```

### Injected Dependencies

Components defined with dependencies receive them in the constructor.

### Accessing Singletons
### getObject(name, ...args)

`getObject` resolves an object from the IoC container. For singletons it returns the shared instance; for prototypes it creates a new instance.

**What `...args` means**
- `...args` are runtime values you pass when resolving an object.
- They are matched by index to `withArgument(index)` in the registration.
- `withArgument(0)` reads the first resolve-time argument, `withArgument(1)` reads the second, and so on.

**Why use `...args`**
- Use them when each created instance needs different data (item component, row component, detail panel).
- This keeps registrations reusable: same prototype registration, different per-instance input.
- It avoids hard-coding values in the container registration when values are only known at runtime.

**When not to use `...args`**
- For shared services/singletons: resolve by name with no runtime args.
- For fixed injected values: prefer `withConstant(...)` at registration time.

Example:
```javascript
// Registration
context.registerPrototype(
  "nodeItem",
  NodeItem,
  cydran.argumentsBuilder()
    .withArgument(0) // item
    .withArgument(1) // selected flag
    .build()
);

// Resolve
const comp = this.$c().getObject("nodeItem", enrichedNode, isSelected);
// NodeItem constructor args:
// arg 0 -> enrichedNode
// arg 1 -> isSelected
```

If you want fixed values, use `withConstant(...)` instead of `withArgument(...)`.

Get registered singletons:

```javascript
onMount() {
  this.app = this.$c().getObject('app');
}
```

---

## Lifecycle Hooks

### Component Lifecycle

#### Lifecycle States

- **Constructed**
  - Entered right after `new YourComponent(...)` and `super(...)` complete.
  - The component instance exists, but it is not mounted in the DOM yet.
  - Best for: defining initial state fields, message subscriptions, and lightweight setup.

- **Mounted**
  - Entered after Cydran has inserted the component DOM and called `onMount()`.
  - The component is active and interactive.
  - Best for: reading services, starting timers/listeners, and loading data that drives UI.

- **Unmounted**
  - Entered when the component is removed from DOM but not disposed.
  - The instance may return later (remount), so treat this as a pause, not destruction.
  - Best for: pausing intervals, detaching external listeners, stopping temporary work.

- **Disposed**
  - Final state after `$release()`/framework teardown.
  - The instance should be treated as dead and must not be reused.
  - Best for: final cleanup of resources that outlive DOM mounting.

#### Overridable Lifecycle Functions

#### `constructor()`
- Called when the component class is instantiated.
- Must call `super(...)` first.
- Use for initial in-memory state and wiring that does not require mounted DOM.

```javascript
constructor() {
  super('<div>{{m().greeting}}</div>');
  this.greeting = 'Hello World!';
}
```

#### `onMount()`
- Called after the component is mounted to the DOM.
- Runs each time the component is mounted (initial mount and remount paths may differ by app flow).
- Use for work that depends on active component context/DOM.

```javascript
onMount() {
  // Acquire services/data, attach listeners, start intervals
}
```

#### `onUnmount()`
- Called when the component is detached from DOM but may be mounted again.
- Use for reversible cleanup (pause polling, unsubscribe temporary listeners).

```javascript
onUnmount() {
  // Stop temporary work that should not run while detached
}
```

#### `onRemount()`
- Called when an unmounted component is attached again.
- Use for re-activation logic (resume timers/listeners, refresh stale values).
- It is only called for detach/attach flows. If a component is disposed (`$release()`), `onRemount()` will not run for that instance.

```javascript
onRemount() {
  // Resume work paused in onUnmount()
}
```

### Lifecycle Guarantees and Idempotency

- `onUnmount()` is not guaranteed to be followed by `onRemount()` for the same instance; teardown paths can go directly to disposal.
- Write `onMount()` and `onRemount()` as idempotent operations so repeated attach cycles do not duplicate listeners/timers/subscriptions.
- Treat `$release()` as final and one-way. After release, do not schedule new work on that instance.

Resource handling guide:

| Resource | Do | Do not |
|----------|----|--------|
| Timers/intervals | Start in `onMount()`/`onRemount()` and clear in `onUnmount()`/`$release()` | Start duplicates on every mount without clearing |
| DOM/event listeners | Attach when mounted; detach in `onUnmount()` | Leave listeners attached while unmounted |
| Messaging subscriptions | Ensure subscriptions are not registered repeatedly; clean up if manually registered | Re-register the same handler each mount |
| Async requests | Guard late responses (`disposed`/`isMounted` checks) and cancel if possible | Mutate state after unmount/dispose without checks |

#### `$release()`
- Final disposal hook during teardown.
- Use for one-time cleanup of long-lived resources.
- After this, assume the component will never be used again.

```javascript
$release() {
  // Final cleanup: close channels, clear references, release handles
}
```

### Hook Order

1. Component instantiation (`constructor`)
2. Component DOM is prepared/inserted
3. `onMount()` on first mount
4. Optional detach/attach cycle: `onUnmount()` then `onRemount()`
5. Final teardown: `$release()`

### Lifecycle State Machine

<svg viewBox="0 0 760 200" width="100%" height="200" role="img" aria-label="Component lifecycle state machine">
  <defs>
    <marker id="arrow" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto">
      <path d="M0,0 L8,3 L0,6 Z" fill="#2d6a4f"></path>
    </marker>
  </defs>
  <circle cx="20" cy="43" r="6" fill="#2d6a4f"></circle>
  <text x="36" y="48" text-anchor="start" font-size="12" fill="#6b5f52">start</text>

  <rect x="70" y="20" width="140" height="46" rx="8" fill="#ffffff" stroke="#e7ded3"></rect>
  <text x="140" y="48" text-anchor="middle" font-size="14" fill="#1e1b16">Constructed</text>

  <rect x="240" y="20" width="140" height="46" rx="8" fill="#ffffff" stroke="#e7ded3"></rect>
  <text x="310" y="48" text-anchor="middle" font-size="14" fill="#1e1b16">Mounted</text>

  <rect x="410" y="20" width="140" height="46" rx="8" fill="#ffffff" stroke="#e7ded3"></rect>
  <text x="480" y="48" text-anchor="middle" font-size="14" fill="#1e1b16">Unmounted</text>

  <rect x="580" y="20" width="140" height="46" rx="8" fill="#ffffff" stroke="#e7ded3"></rect>
  <text x="650" y="48" text-anchor="middle" font-size="14" fill="#1e1b16">Disposed</text>

  <line x1="24" y1="43" x2="70" y2="43" stroke="#2d6a4f" stroke-width="2" marker-end="url(#arrow)"></line>

  <line x1="210" y1="43" x2="240" y2="43" stroke="#2d6a4f" stroke-width="2" marker-end="url(#arrow)"></line>
  <text x="225" y="30" text-anchor="middle" font-size="12" fill="#6b5f52">onMount()</text>

  <line x1="380" y1="43" x2="410" y2="43" stroke="#2d6a4f" stroke-width="2" marker-end="url(#arrow)"></line>
  <text x="395" y="30" text-anchor="middle" font-size="12" fill="#6b5f52">onUnmount()</text>

  <path d="M480,70 C480,120 310,120 310,70" fill="none" stroke="#2d6a4f" stroke-width="2" marker-end="url(#arrow)"></path>
  <text x="395" y="122" text-anchor="middle" font-size="12" fill="#6b5f52">onRemount()</text>

  <line x1="310" y1="10" x2="580" y2="10" stroke="#2d6a4f" stroke-width="2" marker-end="url(#arrow)"></line>
  <text x="445" y="6" text-anchor="middle" font-size="12" fill="#6b5f52">$release()</text>
</svg>

---

## Advanced Features

### Logging

Enable and configure logging:

```javascript
const PROPERTIES = {
  [cydran.PropertyKeys.CYDRAN_LOG_LEVEL]: 'info'
};
```

### Cydran Properties Guide

A Cydran property is a configuration entry in the `properties` object passed to `cydran.create(...)`. Properties control framework/runtime behavior (for example logging and strict-mode behavior) and let you add app-specific configuration keys.

Why you use properties:
- Centralize runtime configuration in one place.
- Toggle diagnostics/safety settings without changing component logic.
- Provide consistent environment-specific behavior (dev/test/prod).

How to apply properties:

```javascript
const PROPERTIES = {
  [cydran.PropertyKeys.CYDRAN_LOG_LEVEL]: "info",
  [cydran.PropertyKeys.CYDRAN_STRICT_ENABLED]: true,
  "app.apiBaseUrl": "/api"
};

const root = cydran.create("#app", PROPERTIES, rootCapability);
```

Common property categories:
- Logging: level, label, appenders, preamble formatting.
- Strict-mode/safety: validation-oriented runtime checks/messages.
- Runtime strategy/tuning: framework behavior toggles exposed by `PropertyKeys`.
- App-specific settings: custom keys your app/services read.

Environment profile examples:

```javascript
const DEV_PROPERTIES = {
  [cydran.PropertyKeys.CYDRAN_LOG_LEVEL]: "debug",
  [cydran.PropertyKeys.CYDRAN_STRICT_ENABLED]: true
};

const PROD_PROPERTIES = {
  [cydran.PropertyKeys.CYDRAN_LOG_LEVEL]: "warn",
  [cydran.PropertyKeys.CYDRAN_STRICT_ENABLED]: false
};
```

Usage notes:
- Treat properties as bootstrap configuration supplied at `cydran.create(...)` time.
- Keep app-specific keys namespaced (for example `app.*`, `inventory.*`).
- Keep a single properties module per environment to avoid scattered config.
- Use `PropertyKeys` constants whenever available to reduce typos.

You can also add custom properties and strict-mode settings:

```javascript
const PROPERTIES = {
  ['todo.person']: '',
  [PropertyKeys.CYDRAN_STRICT_ENABLED]: true,
  [PropertyKeys.CYDRAN_STRICT_STARTPHRASE]: 'Your motivational message',
  [PropertyKeys.CYDRAN_LOG_LEVEL]: 'trace',
  [PropertyKeys.CYDRAN_LOG_LABEL]: 'ctdmvc'
};
```

### Debug Output (Optional)

Logging output is optional and fully controlled by properties. Set a log level in the properties object passed to `cydran.create(...)`:

```javascript
const PROPERTIES = {
  [cydran.PropertyKeys.CYDRAN_LOG_LEVEL]: 'debug'
};
```

Common levels: `trace`, `debug`, `info`, `warn`, `error`, `fatal`, `disabled`.

Notes:
- If logging is too noisy, raise the level (e.g., `warn`) or set `disabled`.
- You can customize preamble order with `CYDRAN_LOG_PREAMBLE_ORDER` (e.g., `time:level:name`).
- If you supply custom appenders, ensure `consoleAppender` is included unless you explicitly suppress it.

Example with warnings only:

```javascript
const PROPERTIES = {
  [cydran.PropertyKeys.CYDRAN_LOG_LEVEL]: 'warn',
  [cydran.PropertyKeys.CYDRAN_LOG_PREAMBLE_ORDER]: 'time:level:name'
};
```

### Logging Profiles and Appender Examples

Recommended defaults by environment:
- Development: `debug` (or `trace` when diagnosing framework flow).
- Test/CI: `info` or `warn` to reduce noise.
- Production: `warn` or `error`.

Labeling guidance:
- Use `CYDRAN_LOG_LABEL` to identify app/module source in shared consoles.
- Use `CYDRAN_LOG_LABEL_VISIBLE` to toggle label display when logs are too dense.

Appender configuration shape example:

```javascript
const PROPERTIES = {
  [cydran.PropertyKeys.CYDRAN_LOG_LEVEL]: "info",
  [cydran.PropertyKeys.CYDRAN_LOG_LABEL]: "inventory",
  [cydran.PropertyKeys.CYDRAN_LOG_LABEL_VISIBLE]: true,
  [cydran.PropertyKeys.CYDRAN_LOG_PREAMBLE_ORDER]: "time:level:name",
  [cydran.PropertyKeys.CYDRAN_LOG_APPENDERS]: [
    "consoleAppender"
  ]
};
```

Strategy/runtime notes:
- `CYDRAN_LOG_STRATEGY` controls how logging is applied internally; keep default unless you have a specific runtime requirement.
- Treat logging properties as bootstrap configuration passed into `cydran.create(...)`.
- For message-heavy debugging, include correlation fields (for example `messageId`, `correlationId`) in log payloads.

### Scope Functions

Register reusable functions accessible from templates via `s()`:

```javascript
function rootCapability(context) {
  context.getScope().add('exclaim', (str) => `${str}!`);
}
```

Usage in template:
```html
{{s().exclaim(m().greeting)}}
```

### Filters

Create computed filtered collections with predicates:

```javascript
this.filtered = this.$c()
  .createFilter('m().messages')
  .withPredicate("v().startsWith('Hello')", 'm().clickCount')
  .build();
```

Then use in template:
```html
<ul c-each="m().filtered.items()">
  <li>{{v()}}</li>
</ul>
```

### Root Initialization

Customize what component loads initially:

```javascript
const root = cydran.create('#app', PROPERTIES, rootCapability);
root.addInitializer(null, s => {
  s.setComponentByObjectId('app');  // Load app component by default
});
root.start();
```

---

## State Machine

Cydran exposes `stateMachineBuilder` for lightweight state machines you can use in your components or services.

### Basic Example

```javascript
const { stateMachineBuilder } = cydran;

const machine = stateMachineBuilder('IDLE')
  .withState('IDLE', [model => { model.status = 'idle'; }])
  .withState('RUNNING', [model => { model.status = 'running'; }])
  .withTransition('IDLE', 'start', 'RUNNING', [
    model => { model.startedAt = Date.now(); }
  ])
  .withTransition('RUNNING', 'stop', 'IDLE', [
    model => { model.stoppedAt = Date.now(); }
  ])
  .build();

const model = { status: 'idle' };
const instance = machine.create(model);

// Trigger transitions
machine.submit('start', instance);
machine.submit('stop', instance);
```

### Guarded Transition Example

```javascript
const machine = stateMachineBuilder('CLOSED')
  .withState('CLOSED', [])
  .withState('OPEN', [])
  .withTransition('CLOSED', 'open', 'OPEN', [
    model => { model.isOpen = true; }
  ], (model) => model.canOpen === true)
  .build();
```

### Notes

- `withState(stateName, [handlers])` registers entry handlers for that state.
- `withTransition(fromState, input, toState, [handlers], [guard])` adds a transition.
- `build()` validates the machine and returns a reusable definition.

### State Machine Semantics

- Treat `machine.create(model)` as creating a machine instance bound to one model object.
- Treat `machine.submit(input, instance)` as a command that attempts one transition for the current state + input.
- If a transition has a guard, it must pass before transition handlers run.
- On a successful transition, run transition side effects and then apply target-state entry behavior.
- If no transition matches (or guard blocks), treat it as a no-op unless your version explicitly throws.
- Keep handlers and guards synchronous. Trigger async work outside the transition path.

### Troubleshooting & Integration

- Invalid transition symptoms: state appears unchanged after `submit(...)`.
- First checks:
  - Confirm the current state actually defines the input.
  - Confirm guard conditions are true for the current model.
  - Confirm state names and input tokens match exactly (case-sensitive).
- Debug pattern:
  - Log before `submit(...)` with current state/input.
  - Log inside transition handlers and state handlers.
  - Log model fields changed by handlers.
- Component integration pattern:
  - Build machine in `constructor()` (or module scope) when definition is static.
  - Create instance per component/service model.
  - Call `submit(...)` from user events/messages, then update/bind UI from model fields.
- API contract caution:
  - Do not depend on `submit(...)` return value unless you verify your exact type definitions (`src/cydran.d.ts` for your build).

### State Management Concepts

Use layered state management so each concern has a clear home.

1. Component state (UI-local):
   - Keep view-specific state on the component model (`this.*`), bound with `m()`.
   - Use for toggles, form fields, selected row, expanded panels, etc.

2. Shared/domain state (cross-component):
   - Keep shared business data in singleton services/repositories.
   - Register with `registerSingleton(...)` and inject into components.

3. Derived/computed state:
   - Use `createFilter(...)` and scope functions for computed collections/values.
   - Avoid duplicating derived arrays/flags in multiple places.

4. Workflow state:
   - Use `stateMachineBuilder` for finite workflows (modes, async phases, multi-step flows).
   - Keep transitions explicit and side effects attached to transitions/states.

5. Event-driven synchronization:
   - Use Pub/Sub messages to notify other components/services of state changes.
   - Prefer narrow propagation scopes and feature-specific channels.

6. Lifecycle-aware handling:
   - `constructor`: initialize in-memory defaults.
   - `onMount`: connect/start work that needs active context.
   - `onUnmount`: pause reversible work.
   - `onRemount`: resume/reactivate.
   - `$release`: final cleanup.

Decision guide:
- UI-only state -> component model.
- Shared business state -> singleton service/repository.
- Derived view state -> filters/scope functions.
- Explicit transitions -> state machine.
- Cross-boundary notifications -> Pub/Sub.

### Operational Guidance

#### Error Handling Contracts

- Document and standardize handling for common runtime failures:
  - Missing object ids/lookup failures.
  - Region update failures (for example locked regions).
  - Invalid state machine transitions/guards.
- Wrap integration boundaries (API calls, message handlers) with explicit error logging and fallback behavior.

#### Performance Guidance

- Prefer targeted watches and predicates; avoid broad/high-frequency expression watching.
- Keep Pub/Sub payloads small and avoid unnecessary global broadcasts.
- For large lists, use stable ids (`c-each-idkey`) and avoid full-list remount patterns.
- Profile noisy flows (messages, digest/watches) before increasing complexity.

#### Testing Patterns

- Unit-test services/repositories independently from UI components.
- For components, test:
  - binding-driven state changes,
  - message send/receive behavior,
  - lifecycle hooks (`onMount`/`onUnmount`/`onRemount`).
- Keep message constants centralized so tests and runtime stay aligned.

#### Migration and Versioning

- Track framework version and maintain a change log for app-level upgrades.
- Verify:
  - DI registrations and resolver behavior,
  - message propagation assumptions,
  - template syntax/binding behavior.
- Prefer incremental upgrades with focused regression tests around lifecycle and messaging.

#### Security and Safety

- Treat user-provided data as untrusted before rendering/binding.
- Keep expressions deterministic; avoid embedding risky side effects in template expressions.
- Validate external payloads before emitting or handling Pub/Sub events.

#### Project Structure Conventions

- Recommended organization for large projects:
  - `modules/<feature>/components`
  - `modules/<feature>/services`
  - `modules/<feature>/templates`
  - `modules/<feature>/messages`
  - `modules/<feature>/register.ts` (module installer)
- Keep module channels/constants namespaced by feature.

#### Error Reference (Common Cases)

| Symptom | Likely Cause | First Fix |
|------|-------------|-----------|
| Region update fails | Region is locked (`lock` or implicit lock) | Remove lock or target a mutable region |
| Component not found by id | Missing/incorrect registration id | Verify `register*` id and `setComponentByObjectId(...)` usage |
| Message not received | Channel/scope mismatch or receiver not subscribed | Check channel names, `To.*` scope, and `subscribe(...)` wiring |
| State transition ignored | No matching transition or guard blocked | Validate current state, input token, and guard condition |

#### Async Patterns

- Keep state-machine guards/handlers synchronous when possible.
- Do async work in lifecycle/message handlers, then update model state deterministically.
- In async callbacks, guard against unmounted/disposed instances before mutating state.
- Prefer publishing completion/error events after async operations instead of coupling callers to internal promise chains.

#### Testing Example (Component + Messaging)

```javascript
// Pseudocode test pattern
const comp = new TodoItem(/* mocked deps */);
comp.onMount();

comp.toggleDone();

// assert model change
expect(comp.done).toBe(true);
// assert outbound message/send side effect
expect(mockTx.send).toHaveBeenCalled();
```

#### Context Hierarchy Example

```javascript
function rootCapability(context) {
  context.registerSingleton("appService", AppService);

  const inventory = context.addChild("inventory");
  inventory.registerSingleton("inventoryRepo", InventoryRepo);
}
```

Use `To.CONTEXT` when a feature should stay inside its child context boundary.

#### Performance Playbook

- High message volume:
  - Narrow scope (`PARENT`/`CONTEXT`) and reduce global broadcasts.
- Slow lists:
  - Ensure stable `c-each-idkey`; avoid full array replacement when partial updates are possible.
- Excess watcher churn:
  - Watch only specific expressions; avoid broad `m()`-level watchers.
- Remount-heavy screens:
  - Make `onMount`/`onRemount` idempotent and avoid duplicate subscriptions/listeners.

#### Migration Checklist

- Pin the current Cydran version before upgrading.
- Re-run tests around:
  - DI registrations/resolvers,
  - Pub/Sub propagation assumptions,
  - lifecycle hook timing,
  - template bindings/markers.
- Upgrade incrementally and capture behavior changes in an app-level migration note.

#### Property Keys Quick Reference

| Key | Purpose | Typical usage |
|-----|---------|---------------|
| `CYDRAN_LOG_LEVEL` | Controls log verbosity | `debug` in dev, `warn/error` in prod |
| `CYDRAN_LOG_LABEL` | Adds app/module label to logs | Multi-module apps and shared console output |
| `CYDRAN_STRICT_ENABLED` | Enables strict validation behavior | Turn on in development |
| `CYDRAN_STRICT_STARTPHRASE` | Strict-mode startup text/message | Team-specific strict prompt text |
| `CYDRAN_LOG_PREAMBLE_ORDER` | Controls log prefix order | Standardize log readability |

For the full list, see `PropertyKeys` in Section 15.

### Behaviors

Behaviors are reusable binding/DOM interaction units used by the framework to implement template attributes and element-level logic.

What they are:
- A behavior encapsulates one focused UI concern (for example propagation handling, expression gating, or value handling).
- Behaviors are applied at bind/render time and operate closer to element/binding semantics than full components.

When to use behaviors vs components:
- Use a component when you need a template + model + lifecycle for a UI block.
- Use a behavior when you need a reusable element/binding rule that can be attached across components.

Related exports in Cydran:
- `Behavior`
- `AbstractBehavior`
- `AbstractValueBehavior`
- `BehaviorFlags`
- `BehaviorAttributeConverters`

Conceptual custom behavior sketch:

```javascript
// Conceptual example only: adapt to the exact signatures in your cydran.d.ts
class MyBehavior extends cydran.AbstractBehavior {
  // lifecycle hooks/methods depend on your framework version
  // implement behavior-specific binding logic here
}
```

Behavior authoring workflow:
- Define the attribute contract (required value, optional flags, defaults).
- Implement minimal bind/update logic in one behavior class.
- Validate incoming attribute values early and fail fast on invalid input.
- Keep DOM mutations localized and reversible when possible.
- Add focused tests for parsing, binding updates, and cleanup semantics.

Best practices:
- Keep each behavior single-purpose and side-effect minimal.
- Validate/normalize attribute values before acting on them.
- Prefer deterministic behavior logic (no hidden async state unless required).
- Document any expected attribute contract (required value format, defaults, flags).

## Library Exports (Constants and Enums)

The Cydran build exposes the following constants/enums for application use:

### `PropertyKeys`

Configuration keys for the properties map passed to `cydran.create(...)`.

```
CYDRAN_CLONE_MAX_EVALUATIONS
CYDRAN_DIGEST_MAX_EVALUATIONS
CYDRAN_EQUALS_MAX_EVALUATIONS
CYDRAN_LAZY_STARTPHRASE
CYDRAN_LOG_COLOR_PREFIX
CYDRAN_LOG_LABEL
CYDRAN_LOG_LABEL_VISIBLE
CYDRAN_LOG_LEVEL
CYDRAN_LOG_APPENDERS
CYDRAN_LOG_ALLOW_SUPRESS_DEFAULT_APPENDER
CYDRAN_LOG_PREAMBLE_ORDER
CYDRAN_LOG_STRATEGY
CYDRAN_OVERRIDE_WINDOW
CYDRAN_STARTUP_SYNCHRONOUS
CYDRAN_STRICT_ENABLED
CYDRAN_STRICT_MESSAGE
CYDRAN_STRICT_STARTPHRASE
CYDRAN_STYLES_ENABLED
```

### `To`

Message propagation targets for transmitters.

| Scope | Description | Typical use |
|------|-------------|-------------|
| `GLOBALLY` | Broadcast across the full component graph. | App-wide notifications and global refresh events. |
| `CONTEXT` | Limit delivery to the current context boundary. | Keep traffic local to a subtree/module. |
| `DESCENDANTS` | Send downward to all nested descendants. | Parent pushes updates to all nested children. |
| `IMMEDIATE_CHILDREN` | Send only to direct child nodes. | Parent coordinates immediate children only. |
| `IMMEDIATE_CHILD_COMPONENTS` | Send to direct child component instances. | Skip deeper descendants and target direct components. |
| `PARENT` | Send to the direct parent only. | Child raises an action/event one level up. |
| `PARENTS` | Bubble upward through all ancestors. | Escalate events to any ancestor listener. |
| `ROOT` | Route to the root component context. | Centralized app-shell handling. |

Usage notes:
- Combine `To.*` with channel filtering for predictable routing.
- `send(propagation, channelName, messageName, payload?, startFrom?)` supports advanced origin control via `startFrom`.

### `Events`

Component lifecycle/mutation events on the internal channel.

```
AFTER_CHILD_ADDED
AFTER_CHILD_CHANGED
AFTER_CHILD_REMOVED
AFTER_PARENT_ADDED
AFTER_PARENT_CHANGED
AFTER_PARENT_REMOVED
BEFORE_CHILD_ADDED
BEFORE_CHILD_CHANGED
BEFORE_CHILD_REMOVED
BEFORE_PARENT_ADDED
BEFORE_PARENT_CHANGED
BEFORE_PARENT_REMOVED
COMPONENT_NESTING_CHANGED
CYDRAN_PREAPP_DISPOSAL
```

### `BehaviorFlags`

Flags used by behaviors.

```
PROPAGATION
EXPRESSION_DISALLOWED
```

### `JSType`

String values representing JavaScript types.

```
STR
BOOL
BIGINT
NUM
SYM
FN
OBJ
UND
```

### `ArgumentType`

Argument descriptors used with the IoC container.

```
OBJECT
PROVIDER
TRANSMITTER
RECEIVER
MESSAGE_SUBSCRIBER
INSTANCE_ID
INSTANCE_ID_PROVIDER
CONTEXT
ARGUMENT
LOGGER
FUNCTION
CONSTANT
PROPERTY
PROPERTY_PROVIDER
PROPERTY_SUBSCRIBER
PROPERTY_FALLBACK_SUBSCRIBER
SCOPE_ITEM
```

---

## Library Exports (Core API)

This project uses the full set of Cydran exports defined in `src/cydran.d.ts`. For completeness, all exported symbols are listed below.

### Classes

- `AbstractBehavior`
- `AbstractValueBehavior`
- `Component`
- `ElementComponent`

### Functions

- `argumentsBuilder`
- `create`
- `enumKeys`
- `getLogger`
- `isDefined`
- `merge`
- `noConflict`
- `overlay`
- `padLeft`
- `padRight`
- `requireNotNull`
- `requireValid`
- `setStrictTypeChecksEnabled`
- `stateMachineBuilder`
- `uuidV4`

### Interfaces and Types

- `ActionContinuation`
- `Actionable`
- `Appender`
- `ArgumentOption`
- `ArgumentsResolvers`
- `ArgumentsResolversBuilder`
- `Attributes`
- `Behavior`
- `Builder`
- `ComponentOptions`
- `Context`
- `DestinationContinuation`
- `DigestableSource`
- `DigestionCandidate`
- `ElementOperations`
- `Evaluatable`
- `Filter`
- `FilterBuilder`
- `ForChannelContinuation`
- `FormOperations`
- `InternalComponentOptions`
- `IntervalContinuation`
- `IsLevelType`
- `LimitOffsetFilter`
- `Logger`
- `Machine`
- `MachineBuilder`
- `MachineState`
- `Mediator`
- `Messagable`
- `MetadataContinuation`
- `MutableProperties`
- `Nestable`
- `Notifyable`
- `OnContinuation`
- `PagedFilter`
- `PropFlagVals`
- `Properties`
- `Receivable`
- `Receiver`
- `RegionContinuation`
- `Register`
- `Releasable`
- `Renderer`
- `Scope`
- `SendContinuation`
- `Sendable`
- `SeriesOperations`
- `Root`
- `Tellable`
- `Watchable`

### Root API (object returned by `create(...)`)

The `create(...)` function returns a `Root`. These are the available methods:

- `getContext()`
- `before()`
- `after()`
- `start()`
- `setComponent(component)`
- `setComponentByObjectId(componentName, defaultComponentName?)`
- `isStarted()`
- `addInitializer(thisObject, callback)`
- `$release()`

### Context API (object returned by `root.getContext()`)

`getContext()` returns a `Context`. These are the available methods:

**Context methods**
- `getChild(name)`
- `hasChild(name)`
- `addChild(name, initializer?)`
- `removeChild(name)`
- `getObject(id, ...instanceArguments)`
- `getProperties()`
- `getScope()`
- `getRoot()`
- `isRoot()`
- `getParent()`
- `registerImplicit(id, template, options?)`
- `getName()`
- `getFullName()`
- `addPreInitializer(thisObject, callback)`
- `addInitializer(thisObject, callback)`
- `addDisposer(thisObject, callback)`
- `configure(callback, thisObject?)`
- `addListener(thisObject, callback)`
- `removeListener(thisObject, callback)`

**Messaging (from Sendable/Tellable/Receivable)**
- `send(propagation, channelName, messageName, payload?, startFrom?)`
- `message(channelName, messageName, payload?)`
- `tell(name, payload?)`

**Registration (from Register)**
- `registerConstant(id, instance)`
- `registerPrototype(id, classInstance, resolvers?, localResolution?)`
- `registerPrototypeWithFactory(id, factoryFn, resolvers?, localResolution?)`
- `registerSingleton(id, classInstance, resolvers?, localResolution?)`
- `registerSingletonWithFactory(id, factoryFn, resolvers?, localResolution?)`
- `hasRegistration(id)`
- `expose(id)`

**Lifecycle (from Releasable)**
- `$release()`

### Type Aliases

- `BehaviorAttributeConverters`
- `BiConsumer`
- `BiPredicate`
- `CallBackThisObject`
- `Consumer`
- `FieldValidations`
- `MessageCallback`
- `Predicate`
- `PropertyChangeCallback`
- `PropertyChangeFallbackCallback`
- `PropertyFallBackSubscriber`
- `PropertyProvider`
- `PropertySubscriber`
- `SimpleMap`
- `Transmitter`
- `Type`
- `VarConsumer`
- `VarPredicate`

These are stable entry points for building and composing components, behaviors, and infrastructure.

## Best Practices

### 1. **Separation of Concerns**

- **Components**: UI logic and state management
- **Services/Repositories**: Data access and business logic
- **Models**: Plain data objects

What to do:
- Keep components focused on UI coordination: bindings, user events, and view state.
- Move business rules/data fetching into injected services or repositories.
- Keep domain objects/model data free of UI framework concerns.
- Publish domain events/messages from components/services instead of directly mutating unrelated components.

Avoid:
- API calls and persistence logic directly inside component event handlers.
- Sharing mutable UI state across components through direct references.
- Mixing template formatting logic with repository/data access logic.

```javascript
// Service: business/data logic
class GreetingService {
  async getGreeting() {
    // fetch or compute domain data here
    return "Hello World!";
  }
}

// Component: UI coordination only
class HelloWorld extends cydran.Component {
  constructor(greetingService) {
    super(template("hello-world"));
    this.greetingService = greetingService;
    this.greeting = "";
  }

  async onMount() {
    this.greeting = await this.greetingService.getGreeting();
  }
}
```

### 2. **Use Message-Based Communication**

Don't pass data directly between components. Use messages instead:

```javascript
// Good: Decoupled through messaging
this.txmitr.send(To.GLOBALLY, APP_CHANNEL, CLICKED, this.clickCount);

this.$c().onMessage(CLICKED)
  .forChannel(APP_CHANNEL)
  .invoke(count => this.clickCount = count);

// Avoid: Tight coupling
otherComponent.updateGreeting();
```

### 3. **Leverage Dependency Injection**

Register services as singletons, components as prototypes:

```javascript
// Singleton - shared across app
context.registerSingleton('app', HelloWorld);
```

### 4. **Watch Expressions Sparingly**

Only watch expressions you need to react to:

```javascript
// Watch the specific property that changes
this.$c().onExpressionValueChange('m().clickCount', () => {
  // React only to click count changes
});

// Don't watch everything
// this.$c().onExpressionValueChange('m()', () => { /* ... */ });
```

### 5. **Use Filters for Computed Collections**

Don't manually filter arrays. Use Cydran filters:

```javascript
// Good: Reactive and efficient
this.filtered = this.$c()
  .createFilter('m().messages')
  .withPredicate("v().includes('clicked')", 'm().clickCount')
  .build();
```

### 6. **Initialize in onMount()**

Defer initialization that requires DOM or services:

```javascript
constructor() {
  super(template('hello-world'));
  // Setup UI model only
  this.greeting = 'Hello World!';
}

onMount() {
  // Get services and load data
}
```

### 7. **Keep Templates Clean**

Use component methods instead of complex logic in templates:

```javascript
// Good: Logic in component
class HelloWorld extends cydran.Component {
  getGreetingText() {
    return `${this.greeting} (${this.clickCount})`;
  }
}

// Template
<span>{{m().getGreetingText()}}</span>

// Avoid: Complex expressions
<span>{{m().greeting}} ({{m().clickCount}})</span>
```

### 8. **Use Strict Mode**

Enable strict mode during development for better validation:

```javascript
const PROPERTIES = {
  [cydran.PropertyKeys.CYDRAN_STRICT_ENABLED]: true,
  [cydran.PropertyKeys.CYDRAN_STRICT_STARTPHRASE]: 'Your motivational message'
};
```

---

## Project Example: Hello World

This project demonstrates a single `HelloWorld` component wired to `#app` in `index.html`.

### Project Structure
```
app.js        # Component + root setup
index.html    # HTML with templates
cydran.js     # Framework build (local)
```

### Run the Example

Open `index.html` in a browser to see the component render and respond to clicks.

---

## Project Example: TodoApp (from todoapp.js)

The TodoApp demonstrates filtered collections, injected dependencies, and message-driven updates.

### Filtered Collection with Parameters

```javascript
this.filtered = this.$c().createFilter('m().todos')
  .withPredicate(
    "p(0) === 'all' || !v().completed && p(0) === 'active' || v().completed && p(0) === 'completed'",
    'm().filterVisiblity'
  )
  .build();
```

### Repository Injection

```javascript
context.registerSingleton(TodoRepo.name, TodoRepo,
  ab().withLogger(`${App.name}.Repo`).withTransmitter().build()
);
```

### Messaging from Items

```javascript
this.txmitr.send(To.GLOBALLY, TODO_CHANNEL, RMV_TODO, this.$c().getValue());
```

### Root Selector

```javascript
const root = create('body>div#appbody', PROPERTIES, rootCapability);
root.addInitializer(null, s => {
  s.setComponentByObjectId('App');
});
```

## Summary

Cydran provides a lightweight, declarative framework for building interactive web applications with:

- **Components** for UI organization
- **Templates** with powerful binding syntax
- **Messaging** for decoupled communication
- **Dependency Injection** for testability and flexibility
- **MVVM Architecture** for clean code organization

By following Cydran's patterns and best practices, you can build maintainable, scalable web applications.

---

## New Project Checklist

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

## Resources

- **GitHub**: https://github.com/cydran/cydran
- **Framework**: An unobtrusive JavaScript presentation framework for the modern web

---

## End-to-End Hello World Example

Below is a complete minimal example. The root component renders in the body and shows a button to reveal a child component. The child says â€œHello Worldâ€ and includes a Hide button that tells the parent to hide it.

**index.html**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Cydran Hello World</title>
</head>
<body>
  <div id="app">cydran failed</div>

  <template id="app-root">
    <div>
      <p>{{m().message}}</p>
      <button c-onclick="m().showChild()" c-enabled="!m().childVisible">Show Hello</button>
      <c-region name="helloRegion"></c-region>
    </div>
  </template>

  <template id="hello-child">
    <div>
      <strong>Hello World</strong>
      <button c-onclick="m().hideMe()">Hide</button>
    </div>
  </template>

  <script src="./cydran.js"></script>
  <script src="./app.js"></script>
</body>
</html>
```

**app.js**
```javascript
const template = (id) => {
  const el = document.querySelector(`template[id="${id}"]`);
  return el ? el.innerHTML.trim() : '';
};

const APP_CHANNEL = 'APP';
const HIDE_CHILD = 'hideChild';

class AppRoot extends cydran.Component {
  constructor() {
    super(template('app-root'));
    this.message = 'Click to reveal the child component.';
    this.childVisible = false;

    this.$c().onMessage(HIDE_CHILD)
      .forChannel(APP_CHANNEL)
      .invoke(() => this.hideChild());
  }

  showChild() {
    if (this.childVisible) {
      return;
    }
    this.$c().regions().setByObjectId('helloRegion', 'helloChild');
    this.childVisible = true;
  }

  hideChild() {
    this.$c().regions().set('helloRegion', null);
    this.childVisible = false;
  }
}

class HelloChild extends cydran.Component {
  constructor() {
    super(template('hello-child'));
  }

  hideMe() {
    this.$c()
      .send(HIDE_CHILD)
      .onChannel(APP_CHANNEL)
      .withPropagation(cydran.To.PARENT);
  }
}

function rootCapability(context) {
  context.registerSingleton('app', AppRoot);
  context.registerPrototype('helloChild', HelloChild);
}

const root = cydran.create('#app', {}, rootCapability);
root.addInitializer(null, s => {
  s.setComponentByObjectId('app');
});
root.start();
```

## A Simple Project

### Basic Setup (This Project)

```javascript
const HELLO_WORLD_TEMPLATE =
  document.querySelector('template[id="hello-world"]')?.innerHTML.trim() ?? '';
const SIMPLE_MESSAGE_TEMPLATE =
  document.querySelector('template[id="simple-message"]')?.innerHTML.trim() ?? '';

const APP_CHANNEL = 'APP';
const HIDE_MESSAGE = 'hideMessage';

class HelloWorld extends cydran.Component {
  constructor() {
    super(HELLO_WORLD_TEMPLATE);

    this.greeting = 'Hello World!';
    this.message = 'Welcome to cydran Framework';
    this.clickCount = 0;
    this.messageShown = false;

    this.$c().onMessage(HIDE_MESSAGE)
      .forChannel(APP_CHANNEL)
      .invoke(() => this.hideMessage());
  }

  updateGreeting() {
    this.clickCount++;
    const messages = {
      1: 'Great! You clicked the button!',
      5: 'Wow, you really like clicking!',
      10: 'You are a master clicker!'
    };
    this.greeting = messages[this.clickCount] ?? this.greeting;
  }

  showMessage() {
    if (this.messageShown) {
      return;
    }

    this.$c().regions().setByObjectId('messageRegion', 'simpleMessage');
    this.messageShown = true;
  }

  hideMessage() {
    this.$c().regions().set('messageRegion', null);
    this.messageShown = false;
  }
}

class SimpleMessage extends cydran.Component {
  constructor() {
    super(SIMPLE_MESSAGE_TEMPLATE);
    this.notice = 'Thanks for trying cydran!';
  }

  hideMe() {
    this.$c()
      .send(HIDE_MESSAGE)
      .onChannel(APP_CHANNEL)
      .withPropagation(cydran.To.PARENT);
  }
}

function rootCapability(context) {
  context.registerSingleton('app', HelloWorld);
  context.registerPrototype('simpleMessage', SimpleMessage);
}

const root = cydran.create('#app', {}, rootCapability);
root.addInitializer(null, s => {
  s.setComponentByObjectId('app');
});
root.start();
```




