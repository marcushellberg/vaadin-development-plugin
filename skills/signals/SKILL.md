---
name: signals
description: >
  Guide Claude on using Vaadin Signals for reactive state management in Vaadin 25 Flow.
  This skill should be used when the user asks to "use signals", "manage state reactively",
  "share state between users", "use reactive state", "use ValueSignal",
  "use NumberSignal", "use ListSignal", "use computed signals",
  "use ComponentEffect", "bind signals to components", or needs help with
  reactive UI updates, signal transactions, signal factories, or
  thread-safe state management in Vaadin Flow.
version: 0.1.0
---

# Reactive State Management with Signals in Vaadin 25

Use the Vaadin MCP tools (`search_vaadin_docs`) to look up the latest documentation whenever uncertain about a specific API detail. Always set `vaadin_version` to `"25"` and `ui_language` to `"java"`.

## What Signals Are

Signals are a reactive state management system for Vaadin Flow. A signal holds a value, and when that value changes, all dependent parts of the UI automatically update — without manually adding and removing change listeners.

Key properties:
- **Reactive** — changes propagate automatically to dependent UI
- **Thread-safe** — designed for concurrent access from multiple users
- **Immutable values** — signals work best with immutable types (String, Integer, Java Records)
- **Transactional** — multiple signal updates can be grouped atomically
- **Hierarchical** — signals can represent complex data structures (lists, maps, trees)

## Core Concepts

### ValueSignal — a single value

```java
ValueSignal<String> name = new ValueSignal<>(String.class);
name.value("John");           // set value
String current = name.value(); // get value
name.update(n -> n + " Doe"); // atomic update based on current value
```

### NumberSignal — numeric values with atomic arithmetic

```java
NumberSignal counter = new NumberSignal();
counter.value(5);             // set
counter.incrementBy(1);       // atomic increment
counter.incrementBy(-2);      // atomic decrement
int count = counter.valueAsInt();
```

### ListSignal — ordered collection

```java
ListSignal<Person> people = new ListSignal<>(Person.class);
people.insertFirst(new Person("Jane", 25));
people.insertLast(new Person("John", 30));

List<ValueSignal<Person>> list = people.value();
list.get(0).value(new Person("Updated", 26)); // update individual item
```

### MapSignal — key-value pairs

```java
MapSignal<String> config = new MapSignal<>(String.class);
config.put("theme", "dark");
config.putIfAbsent("language", "en");

Map<String, ValueSignal<String>> map = config.value();
```

### ReferenceSignal — single-UI mutable state

For state used by only one UI instance (not shared). No thread safety overhead, but also no cross-user sharing:

```java
ReferenceSignal<User> userSignal = new ReferenceSignal<>();
userSignal.value(new User("Jane", 25));
userSignal.modify(user -> user.setAge(26)); // mutable update allowed
```

ReferenceSignal cannot be used in transactions.

## Effects — Reactive UI Updates

Effects are callbacks that re-run automatically when any signal they read changes. They are the bridge between signals and the UI.

### ComponentEffect.effect — bind to a component's lifecycle

```java
ComponentEffect.effect(span, () -> {
    span.setText(firstName.value() + " " + lastName.value());
});
```

The effect:
- Automatically tracks which signals are read inside the callback
- Re-runs when any of those signals change
- Is active only while the component is attached to the UI
- Cleans up automatically when the component is detached

### ComponentEffect.bind — shorthand for simple bindings

```java
ComponentEffect.bind(label, nameSignal, Span::setText);
ComponentEffect.bind(button, enabledSignal, Button::setEnabled);
ComponentEffect.bind(div, visibleSignal, Div::setVisible);
```

Equivalent to `ComponentEffect.effect` but more concise for single-signal, single-property bindings.

### Computed Signals — derived values

Computed signals derive their value from other signals and update automatically:

```java
Signal<String> fullName = Signal.computed(() -> {
    return firstName.value() + " " + lastName.value();
});
```

### Signal Mapping — transform a single signal

Shorthand for a computed that depends on exactly one signal:

```java
Signal<String> ageCategory = ageSignal.map(age ->
    age < 18 ? "Child" : (age < 65 ? "Adult" : "Senior")
);
```

## Signal Factory

Use `SignalFactory` to create signal instances. The factory determines how signals are scoped:

### IN_MEMORY_SHARED — shared across the application

Same signal instance for the same name within the same JVM. Use for state visible to all users:

```java
NumberSignal globalCounter = SignalFactory.IN_MEMORY_SHARED.number("counter");
ValueSignal<String> sharedMessage = SignalFactory.IN_MEMORY_SHARED.value("msg", String.class);
NodeSignal sharedData = SignalFactory.IN_MEMORY_SHARED.node("data");
```

### IN_MEMORY_EXCLUSIVE — per-instance

Always creates a new instance. Use for component-local state:

```java
NumberSignal localCounter = SignalFactory.IN_MEMORY_EXCLUSIVE.number("counter");
```

## Transactions

Group multiple signal updates into an atomic operation. Observers see either all changes or none:

```java
Signal.runInTransaction(() -> {
    firstName.value("John");
    lastName.value("Doe");
    age.value(30);
});
```

## Element Binding

Signals can bind directly to `Element` properties, attributes, text, classes, and styles:

### Text binding

```java
Signal<String> text = counter.map(v -> String.format("Count: %.0f", v));
span.getElement().bindText(text);
```

### Property binding

```java
span.getElement().bindProperty("hidden", hiddenSignal);
```

Supports String, Boolean, Double, JSON, Object (bean), List, and Map types.

### Attribute binding

```java
span.getElement().bindAttribute("hidden",
    hiddenSignal.map(h -> h ? "" : null));
```

### Class binding

```java
span.getElement().getClassList().bind("active", isActiveSignal);
```

### Style binding

```java
span.getElement().getStyle().bind("color", colorSignal);
```

**Important:** while a signal is bound to an element property, manual changes to that property throw `BindingActiveException`. Unbind first with `bindText(null)`, `bindProperty(null)`, etc.

## Complete Example: Shared Counter

```java
public class SharedCounter extends VerticalLayout {
    private final NumberSignal counter =
            SignalFactory.IN_MEMORY_SHARED.number("counter");

    public SharedCounter() {
        Button button = new Button();
        button.addClickListener(click -> counter.incrementBy(1));
        add(button);

        ComponentEffect.effect(button,
            () -> button.setText(
                String.format("Clicked %.0f times", counter.value())));
    }
}
```

All users see the same counter value, and clicking in any browser updates all connected UIs (requires Push to be enabled).

## Best Practices

1. **Use immutable values** — Strings, primitives, Java Records. Mutating an object directly won't trigger reactivity. Always create a new value:

   ```java
   // GOOD: new immutable object
   user.update(u -> new User(u.getName(), u.getAge() + 1));

   // BAD: mutating in place (won't trigger updates!)
   User u = user.value();
   u.setAge(u.getAge() + 1);
   ```

2. **Use `ComponentEffect.effect` for UI bindings** — it handles lifecycle (attach/detach) automatically, preventing memory leaks.

3. **Use `ComponentEffect.bind` for simple one-signal bindings** — cleaner than writing a full effect for `label.setText(signal.value())`.

4. **Use transactions for multi-signal updates** — prevents observers from seeing partial state.

5. **Use `update()` for atomic read-modify-write** — `counter.update(c -> c + 1)` is atomic; reading and then setting is not.

6. **Don't modify signals inside effects or computed callbacks** — they run in read-only transactions. If you must, use `Signal.runWithoutTransaction()`, but be very careful about infinite loops.

7. **Use `peek()` to read without tracking** — inside an effect, `signal.peek()` reads the value without creating a dependency, so the effect won't re-run when that signal changes.

8. **Enable Push for shared signals** — when using `IN_MEMORY_SHARED`, enable server push so changes propagate immediately to all connected UIs.

9. **Prefer `ReferenceSignal` for single-UI mutable state** — no thread-safety overhead when state doesn't need to be shared.
