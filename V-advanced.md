# V. Advanced

## 13. Logging & Properties

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

Customize what component loads initially:

```javascript
const root = cydran.create('#app', PROPERTIES, rootCapability);
root.addInitializer(null, s => {
  s.setComponentByObjectId('app');  // Load app component by default
});
root.start();
```

---

_Property and behavior constants are detailed in Section 15 below._

## 14. Scope Functions, Filters, State Machines

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

## 15. Library Exports (PropertyKeys, To, Events, BehaviorFlags, full API classes/interfaces)

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

```
GLOBALLY
CONTEXT
DESCENDANTS
IMMEDIATE_CHILDREN
IMMEDIATE_CHILD_COMPONENTS
PARENT
PARENTS
ROOT
```

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

