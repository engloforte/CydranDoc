# III. Building Blocks

## 6. Components (creation, options, $c() API, Regions)

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

## 7. Templates (sources, syntax, one-root rule, SVG/renderers)

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
Two-way binding keeps the field value and the UI control in sync in both directions:
- Model -> View: when `m().greeting` changes in code, the `<input>` value updates on the next digest.
- View -> Model: when the user types into the `<input>`/`<textarea>`, Cydran writes the new value back to the bound model property.

This is different from interpolation (`{{ ... }}`), which only pushes model values into the DOM and does not write user edits back to the model.
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

## 8. Custom Elements & Markers (`<c-region>`, `<c-series>`, `<c-component-styles>`)

Cydran uses a few structural markers/placeholders in the DOM. These are normal and help the framework manage dynamic insertion/removal.

- **`<c-region>`**: A placeholder tag that is replaced by the mounted component for that region.
- **`<c-series>`**: A placeholder tag that is replaced by a series insertion point.

### Tip for Debugging: `data-template-id`

Adding `data-template-id="..."`can aid DOM inspection.

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

## 9. Data Binding (expressions, c-model, c-on*, c-if, c-each, c-class, etc.)

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
Two-way binding keeps the field value and the UI control in sync in both directions:
- Model -> View: when `m().greeting` changes in code, the `<input>` value updates on the next digest.
- View -> Model: when the user types into the `<input>`/`<textarea>`, Cydran writes the new value back to the bound model property.

This is different from interpolation (`{{ ... }}`), which only pushes model values into the DOM and does not write user edits back to the model.
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

