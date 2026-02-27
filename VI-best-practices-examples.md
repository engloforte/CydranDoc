# VI. Best Practices & Examples

## 16. Best Practices

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

## 17. Hello World (single & end-to-end)

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

## 18. TodoApp patterns

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

