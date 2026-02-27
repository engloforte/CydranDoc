# IV. Architecture

## 10. Messaging & Communication (channels, send/receive, propagation)

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

### Propagation Scopes (`To.*`)

Use propagation to control how far a message travels from the current component context.

| Scope | Description | Typical use |
|------|-------------|-------------|
| `To.GLOBALLY` | Broadcast across the full component graph. | App-wide notifications and global refresh events. |
| `To.CONTEXT` | Limit delivery to the current context boundary. | Keep traffic local to a subtree/module. |
| `To.DESCENDANTS` | Send downward to all nested descendants. | Parent pushes updates to all nested children. |
| `To.IMMEDIATE_CHILDREN` | Send only to direct child nodes. | Parent coordinates immediate children only. |
| `To.IMMEDIATE_CHILD_COMPONENTS` | Send to direct child component instances. | Skip deeper descendants and target direct components. |
| `To.PARENT` | Send to the direct parent only. | Child raises an action/event one level up. |
| `To.PARENTS` | Bubble upward through all ancestors. | Escalate events to any ancestor listener. |
| `To.ROOT` | Route to the root component context. | Centralized app-shell handling. |

Notes:
- Pair propagation with `.onChannel(...)` / `.forChannel(...)` so only intended subscribers receive the message.
- `send(..., startFrom?)` can change the origin point for propagation in advanced scenarios.

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

## 11. Dependency Injection (registerSingleton/Prototype, ArgumentsBuilder, getObject)

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

## 12. Lifecycle Hooks (constructor -> onMount -> onUnmount -> onRemount -> $release)

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

