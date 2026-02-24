# Signals Patterns Reference

## Signal Types Quick Reference

| Type | Purpose | Key Methods |
|------|---------|-------------|
| `ValueSignal<T>` | Single value of any type | `value()`, `value(T)`, `update(fn)` |
| `NumberSignal` | Numeric with atomic math | `value()`, `value(double)`, `incrementBy()`, `valueAsInt()` |
| `ListSignal<T>` | Ordered collection | `insertFirst()`, `insertLast()`, `value()` → `List<ValueSignal<T>>` |
| `MapSignal<T>` | Key-value pairs | `put()`, `putIfAbsent()`, `value()` → `Map<String, ValueSignal<T>>` |
| `NodeSignal` | Tree structure | `putChildWithValue()`, `insertChildWithValue()`, `asMap()` |
| `ReferenceSignal<T>` | Single-UI mutable state | `value()`, `value(T)`, `modify(Consumer)` |

## Signal Factory Comparison

| Factory | Scope | Use Case |
|---------|-------|----------|
| `IN_MEMORY_SHARED` | Same instance per name/JVM | Shared counters, collaborative data, global config |
| `IN_MEMORY_EXCLUSIVE` | New instance every time | Component-local state, non-shared data |

## Effect Binding Patterns

### Full effect — complex logic

```java
ComponentEffect.effect(component, () -> {
    String first = firstName.value();
    String last = lastName.value();
    boolean hasName = !first.isEmpty() || !last.isEmpty();

    nameLabel.setText(first + " " + last);
    nameLabel.setVisible(hasName);
    placeholder.setVisible(!hasName);
});
```

### Bind shorthand — single signal → single property

```java
ComponentEffect.bind(label, nameSignal, Span::setText);
ComponentEffect.bind(button, enabledSignal, Button::setEnabled);
ComponentEffect.bind(div, nameSignal.map(n -> !n.isEmpty()), Div::setVisible);
```

### Element binding — direct DOM binding

```java
// Text
element.bindText(signal.map(v -> "Value: " + v));

// Attribute
element.bindAttribute("title", tooltipSignal);

// Property
element.bindProperty("hidden", hiddenSignal);

// CSS class
element.getClassList().bind("active", isActiveSignal);

// Style
element.getStyle().bind("color", colorSignal);
```

## Complete Patterns

### Shared todo list

```java
public class TodoList extends VerticalLayout {
    private final ListSignal<String> todos =
            SignalFactory.IN_MEMORY_SHARED.list("todos", String.class);

    public TodoList() {
        TextField input = new TextField("New todo");
        Button addBtn = new Button("Add", e -> {
            if (!input.getValue().isBlank()) {
                todos.insertLast(input.getValue());
                input.clear();
            }
        });

        VerticalLayout list = new VerticalLayout();
        list.setSpacing(false);
        list.setPadding(false);

        ComponentEffect.effect(list, () -> {
            list.removeAll();
            for (ValueSignal<String> todo : todos.value()) {
                HorizontalLayout row = new HorizontalLayout();
                Span text = new Span();
                ComponentEffect.bind(text, todo, Span::setText);
                Button removeBtn = new Button("×", e ->
                    todo.getParent().remove(todo.id()));
                row.add(text, removeBtn);
                list.add(row);
            }
        });

        add(new HorizontalLayout(input, addBtn), list);
    }
}
```

### Form with signal-backed state

```java
public class ProfileEditor extends VerticalLayout {
    private final ValueSignal<String> firstName =
            SignalFactory.IN_MEMORY_EXCLUSIVE.value("first", String.class);
    private final ValueSignal<String> lastName =
            SignalFactory.IN_MEMORY_EXCLUSIVE.value("last", String.class);
    private final Signal<String> fullName = Signal.computed(() ->
            firstName.value() + " " + lastName.value());

    public ProfileEditor() {
        TextField firstField = new TextField("First Name");
        TextField lastField = new TextField("Last Name");
        Span preview = new Span();

        // Two-way binding (field → signal)
        firstField.addValueChangeListener(e -> {
            if (!e.getValue().equals(firstName.peek())) {
                firstName.value(e.getValue());
            }
        });
        lastField.addValueChangeListener(e -> {
            if (!e.getValue().equals(lastName.peek())) {
                lastName.value(e.getValue());
            }
        });

        // Signal → field
        ComponentEffect.bind(firstField, firstName, TextField::setValue);
        ComponentEffect.bind(lastField, lastName, TextField::setValue);

        // Computed → display
        ComponentEffect.bind(preview, fullName, Span::setText);

        add(firstField, lastField, preview);
    }
}
```

### Read-only signals for encapsulation

```java
public class CounterService {
    private final NumberSignal counter =
            SignalFactory.IN_MEMORY_SHARED.number("counter");

    public NumberSignal getReadOnlyCounter() {
        return counter.asReadonly();
    }

    public void increment() {
        counter.incrementBy(1);
    }
}
```

## Anti-patterns

### Mutating objects in place

```java
// BAD: won't trigger reactivity
User u = userSignal.value();
u.setName("New Name");  // mutation not detected!

// GOOD: create new immutable value
userSignal.update(u -> new User("New Name", u.getAge()));
```

### Modifying signals inside effects

```java
// BAD: throws exception (read-only transaction)
ComponentEffect.effect(span, () -> {
    otherSignal.value(someSignal.value());  // ERROR!
});

// GOOD: use computed signal instead
Signal<String> derived = Signal.computed(() -> someSignal.value());
```

### Forgetting to check for equality in two-way binding

```java
// BAD: infinite loop risk
field.addValueChangeListener(e -> signal.value(e.getValue()));
ComponentEffect.bind(field, signal, TextField::setValue);

// GOOD: check before updating
field.addValueChangeListener(e -> {
    if (!e.getValue().equals(signal.peek())) {
        signal.value(e.getValue());
    }
});
```

### Using shared signals without Push

```java
// BAD: other users won't see updates until they interact
NumberSignal shared = SignalFactory.IN_MEMORY_SHARED.number("counter");

// GOOD: enable Push in your application
// @Push annotation on AppShellConfigurator implementation
```
