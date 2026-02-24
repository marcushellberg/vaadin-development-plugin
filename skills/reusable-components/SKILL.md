---
name: reusable-components
description: >
  Guide Claude on composing reusable UI components in Vaadin 25 Flow with clean APIs.
  This skill should be used when the user asks to "create a reusable component",
  "build a custom component", "compose components", "extend a component",
  "use Composite", "design a component API", "implement HasValue",
  "implement HasComponents", or needs guidance on component encapsulation,
  when to extend vs compose, and defining clean public interfaces for
  Vaadin Flow components.
version: 0.1.0
---

# Composing Reusable UI Components in Vaadin 25

Use the Vaadin MCP tools (`search_vaadin_docs`, `get_component_java_api`) to look up the latest documentation whenever uncertain about a specific API detail. Always set `vaadin_version` to `"25"` and `ui_language` to `"java"`.

## The Two Approaches: Extend vs. Composite

There are two main ways to create custom components in Vaadin Flow. Choose based on how much of the root component's API you want to expose.

### Approach 1: Extend an existing component

Use when you want to **add** to an existing component's API. The parent class's full public API remains accessible to users of your component.

```java
public class PrimaryButton extends Button {
    public PrimaryButton(String text) {
        super(text);
        addThemeVariants(ButtonVariant.LUMO_PRIMARY);
    }
}
```

Good for: pre-configured variants, adding convenience methods, specializing behavior.

Risk: every public method on the parent is part of your API. Users can call anything on Button, which may break your component's invariants.

### Approach 2: Use Composite<T> (recommended for most cases)

Use when you want to **hide** the root component's API and expose only what you explicitly define. `Composite<T>` wraps a root component and makes `getContent()` protected, so users can only interact through your public methods.

```java
public class UserCard extends Composite<HorizontalLayout> {

    private final Avatar avatar = new Avatar();
    private final Span name = new Span();
    private final Span role = new Span();

    public UserCard() {
        VerticalLayout info = new VerticalLayout(name, role);
        info.setSpacing(false);
        info.setPadding(false);

        getContent().setAlignItems(FlexComponent.Alignment.CENTER);
        getContent().add(avatar, info);
    }

    public void setUser(String userName, String userRole, String imageUrl) {
        name.setText(userName);
        role.setText(userRole);
        avatar.setImage(imageUrl);
        avatar.setName(userName);
    }
}
```

Good for: compound components, encapsulated UI blocks, anything that combines multiple components into a single reusable unit.

**Prefer Composite for new reusable components.** It produces cleaner APIs and prevents accidental misuse.

## Designing a Clean Component API

### Principles

1. **Expose intent, not implementation** — public methods should describe what the component does, not how it's built. `setUser(name, role)` is better than exposing the internal `Span` and `Avatar`.

2. **Make invalid states unrepresentable** — if your component requires both a title and an icon, take them as constructor parameters rather than offering separate setters that can be called in any order.

3. **Follow Vaadin conventions** — users expect familiar patterns:
   - `setValue()` / `getValue()` for components with a value
   - `setLabel()` for field labels
   - `setEnabled()` / `setReadOnly()` for interaction states
   - `addXxxListener()` for events

4. **Use typed events** — define custom `ComponentEvent` subclasses instead of generic callbacks. This integrates with Vaadin's event system and supports `@DomEvent` for client-side events.

### Constructor Design

Provide a no-arg constructor for compatibility with Vaadin's declarative systems, then offer convenience constructors for common usage:

```java
public class StatusBadge extends Composite<Span> {

    public StatusBadge() {
        // default state
    }

    public StatusBadge(Status status) {
        setStatus(status);
    }

    public void setStatus(Status status) {
        getContent().setText(status.getLabel());
        getContent().getElement().getThemeList().clear();
        getContent().getElement().getThemeList().add("badge " + status.getTheme());
    }
}
```

## Implementing Key Interfaces

### HasValue<E, V> — for field-like components

Implement when your component represents an editable value that should work with `Binder`. This is the most important interface for form integration.

Requirements: define `setValue()`, `getValue()`, value change events, empty value, read-only mode, and required indicator.

Extend `AbstractField<C, V>` for a base implementation that handles most of the boilerplate.

```java
public class StarRating extends AbstractField<StarRating, Integer> {

    public StarRating() {
        super(0); // default/empty value
        // build UI: 5 clickable star icons
    }

    @Override
    protected void setPresentationValue(Integer value) {
        // update the star icons to reflect the value
    }
}
```

### HasComponents — for container components

Implement when your component can accept arbitrary child components. Provides `add()`, `remove()`, `removeAll()`.

```java
@Tag("div")
public class CardGroup extends Component implements HasComponents {
    // add() and remove() are provided by the interface
}
```

Only implement this when arbitrary children make sense. If your component has specific slots (e.g., a header and a body), use explicit methods instead:

```java
public void setHeader(Component header) { ... }
public void setBody(Component body) { ... }
```

### HasStyle — for style customization

Automatically available on components that extend `Component`. Provides `addClassName()`, `getStyle()`, etc. Consider whether to delegate or restrict style access in your Composite.

## Composition Patterns

### Pattern: Wrapper with slots

A common pattern for layout-style components with named areas:

```java
public class PageHeader extends Composite<HorizontalLayout> {

    private final Div titleSlot = new Div();
    private final Div actionsSlot = new Div();

    public PageHeader() {
        getContent().setWidthFull();
        getContent().setAlignItems(FlexComponent.Alignment.CENTER);
        getContent().setJustifyContentMode(FlexComponent.JustifyContentMode.BETWEEN);
        getContent().add(titleSlot, actionsSlot);
    }

    public void setTitle(String title) {
        titleSlot.removeAll();
        titleSlot.add(new H2(title));
    }

    public void setActions(Component... actions) {
        actionsSlot.removeAll();
        HorizontalLayout actionBar = new HorizontalLayout(actions);
        actionsSlot.add(actionBar);
    }
}
```

### Pattern: Configurable via builder or fluent API

For components with many optional configuration properties:

```java
public class DataCard extends Composite<VerticalLayout> {

    public DataCard withTitle(String title) {
        // add title component
        return this;
    }

    public DataCard withIcon(VaadinIcon icon) {
        // add icon
        return this;
    }

    public DataCard withValue(String value) {
        // add value display
        return this;
    }
}

// Usage:
new DataCard()
    .withTitle("Revenue")
    .withIcon(VaadinIcon.DOLLAR)
    .withValue("$1.2M");
```

### Pattern: Event-driven communication

Use custom events for parent-child communication instead of callbacks:

```java
public class DeleteConfirmation extends Composite<Div> {

    public Registration addConfirmListener(
            ComponentEventListener<ConfirmEvent> listener) {
        return addListener(ConfirmEvent.class, listener);
    }

    public static class ConfirmEvent extends ComponentEvent<DeleteConfirmation> {
        public ConfirmEvent(DeleteConfirmation source, boolean fromClient) {
            super(source, fromClient);
        }
    }
}
```

## Lifecycle Considerations

- **onAttach()** — called when the component is added to the UI. Use for initialization that requires the component to be in the DOM (e.g., accessing session data, subscribing to event buses).
- **onDetach()** — called before removal from the UI. Use for cleanup (e.g., unsubscribing from event buses, releasing resources).
- Always clean up in `onDetach()` what you set up in `onAttach()` to prevent memory leaks.

## Best Practices

1. **Prefer Composite over direct extension** — it gives you API control and prevents leaking internals.
2. **Keep components focused** — a component should do one thing well. If it has too many responsibilities, split it.
3. **Use typed events over callbacks** — `addXxxListener()` returning `Registration` is the Vaadin way. It supports easy unsubscription and integrates with the event system.
4. **Don't expose internal components** — return data from getters, not the underlying Spans and Divs. If you must expose a sub-component, document that it's an advanced/escape-hatch API.
5. **Document your public API** — Javadoc on public methods of reusable components is worth the effort. Other developers (and AI assistants) will use it.
6. **Test components in isolation** — reusable components should be testable with Vaadin's UI unit testing framework without needing a full application context.
