# Reusable Component Patterns Reference

## Decision Matrix: Which Base to Use?

| Scenario | Base class | Why |
|----------|-----------|-----|
| Pre-configured variant of existing component | Extend the component | You want the full parent API |
| Combining multiple components into one | `Composite<T>` | Hides root API, clean encapsulation |
| Wrapping a web component with a Java API | Extend `Component` with `@Tag` | Direct element access needed |
| Field that works with Binder | `AbstractField<C, V>` | Handles value change boilerplate |
| Container that accepts children | `Component` + `HasComponents` | Provides standard add/remove API |
| Layout-like component with named slots | `Composite<T>` with slot methods | Explicit API over generic add() |

## Composite<T> Template

```java
public class MyComponent extends Composite<VerticalLayout> {

    // Declare internal components as fields
    private final H3 title = new H3();
    private final Span description = new Span();

    public MyComponent() {
        // Configure the root layout
        getContent().setPadding(true);
        getContent().setSpacing(false);
        getContent().add(title, description);
    }

    // Public API — what users of this component can do
    public void setTitle(String text) {
        title.setText(text);
    }

    public void setDescription(String text) {
        description.setText(text);
    }
}
```

Key points:
- `getContent()` returns the root component (protected, not public)
- Internal components are private fields
- Public methods define the component's contract

## AbstractField<C, V> Template

```java
public class RangeSlider extends AbstractField<RangeSlider, Range> {

    public RangeSlider() {
        super(Range.empty());  // default/empty value
        // Build the UI elements
    }

    @Override
    protected void setPresentationValue(Range value) {
        // Update the visual representation
        // Called by setValue() after validation
    }

    // Optional: customize equality check
    @Override
    protected boolean valueEquals(Range value1, Range value2) {
        return Objects.equals(value1, value2);
    }
}
```

## Custom Event Template

```java
// Define the event
public class SelectionEvent extends ComponentEvent<MyComponent> {

    private final String selectedItem;

    public SelectionEvent(MyComponent source, boolean fromClient, String selectedItem) {
        super(source, fromClient);
        this.selectedItem = selectedItem;
    }

    public String getSelectedItem() {
        return selectedItem;
    }
}

// In the component — fire and expose listener registration
public class MyComponent extends Composite<Div> {

    public Registration addSelectionListener(
            ComponentEventListener<SelectionEvent> listener) {
        return addListener(SelectionEvent.class, listener);
    }

    private void onItemClicked(String item) {
        fireEvent(new SelectionEvent(this, false, item));
    }
}
```

## Web Component Wrapper Template

> For in-depth guidance on integrating third-party Web Components or React components from npm, see the `third-party-components` skill.

For wrapping a client-side web component with a Java API:

```java
@Tag("my-web-component")
@JsModule("my-web-component/my-web-component.js")
public class MyWebComponent extends Component {

    // Property access
    public void setLabel(String label) {
        getElement().setProperty("label", label);
    }

    public String getLabel() {
        return getElement().getProperty("label", "");
    }

    // Event handling
    @DomEvent("item-click")
    public static class ItemClickEvent extends ComponentEvent<MyWebComponent> {
        private final String detail;

        public ItemClickEvent(MyWebComponent source, boolean fromClient,
                @EventData("event.detail") String detail) {
            super(source, fromClient);
            this.detail = detail;
        }

        public String getDetail() {
            return detail;
        }
    }

    public Registration addItemClickListener(
            ComponentEventListener<ItemClickEvent> listener) {
        return addListener(ItemClickEvent.class, listener);
    }
}
```

## Lifecycle Pattern: Event Bus Subscription

```java
public class NotificationBell extends Composite<Div> {

    private Registration busRegistration;

    @Override
    protected void onAttach(AttachEvent event) {
        var eventBus = event.getSession().getAttribute(AppEventBus.class);
        busRegistration = eventBus.subscribe(NotificationEvent.class, this::onNotification);
    }

    @Override
    protected void onDetach(DetachEvent event) {
        if (busRegistration != null) {
            busRegistration.remove();
            busRegistration = null;
        }
    }

    private void onNotification(NotificationEvent event) {
        // Update badge count, etc.
    }
}
```

## Anti-patterns

### Leaking internals

```java
// BAD: exposes the internal layout
public class UserCard extends Composite<HorizontalLayout> {
    public HorizontalLayout getLayout() {
        return getContent(); // Don't expose this!
    }
}

// GOOD: expose intent-based methods
public class UserCard extends Composite<HorizontalLayout> {
    public void setCompact(boolean compact) {
        getContent().setSpacing(!compact);
        getContent().setPadding(!compact);
    }
}
```

### Over-extending

```java
// BAD: extending VerticalLayout just to group components
public class LoginForm extends VerticalLayout {
    // Users can call add(), remove(), setSpacing(), etc.
    // which may break the form's layout
}

// GOOD: use Composite to control the API
public class LoginForm extends Composite<VerticalLayout> {
    // Only your public methods are accessible
}
```

### God components

```java
// BAD: one component doing everything
public class UserManagementPanel extends Composite<Div> {
    // user list + user form + role editor + permission matrix + audit log
    // This should be 5+ separate components
}
```

Split into focused components and compose them at the view level.
