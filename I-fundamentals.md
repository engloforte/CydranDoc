# I. Fundamentals

## 1. Overview

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

## 2. Core Concepts (MVVM, Components, Non-Cydran Components, Two-Way Binding)

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

### Access to Non-Cydran Components

- Register them in `rootCapability(context)` as singleton/prototype.
- Resolve them from components with `$c().getObject(...)`.
- Keep UI state in components; keep domain/data logic in these objects.

### PubSub

Non-Cydran Component classes are not auto-subscribed to messaging. To enable PubSub, inject messaging dependencies during registration.

Register with an `ArgumentsBuilder`:

```javascript
const ab = cydran.argumentsBuilder;

context.registerSingleton(
  "apiBroker",
  ApiBroker,
  ab()
    .with("apiService")
    .withReceiver()
    .withTransmitter()
    .withMessageSubscriber()
    .build()
);
```

Wire subscription in the constructor:

```javascript
class ApiBroker {
  constructor(apiService, rx, tx, subscribe) {
    this.apiService = apiService;
    this.rx = rx;
    this.tx = tx;
    subscribe(this.rx, this.rx.message);
  }

  message(message) {
    // handle inbound messages
  }

  publishUpdate(payload) {
    this.tx.send(cydran.To.GLOBALLY, "API_CHANNEL", "API_UPDATED", payload);
  }
}
```

What `subscribe(...)` does:
- Registers this instance's receiver with Cydran's broker.
- Connects inbound broker messages to `rx.message`.
- Makes the instance an active Pub/Sub receiver.
- Without it, the instance is not subscribed to inbound broker delivery.

Quick rules:
- Publish only: inject `withTransmitter()`.
- Publish + subscribe: inject `withReceiver()`, `withTransmitter()`, and `withMessageSubscriber()`, then call `subscribe(...)` in the constructor.
- Use consistent channel/message names and small payload objects.

### Two-Way Data Binding

Changes to model properties automatically update the view, and user interactions update the model.

---
