---
name: ui-unit-testing
description: >
  Guide Claude on writing fast, browser-free UI unit tests for Vaadin 25 Flow views.
  This skill should be used when the user asks to "write a UI test", "unit test a view",
  "test without a browser", "use UIUnitTest", "test a Vaadin component",
  "browser-free testing", "browser-less testing", or needs help with the
  TestBench UI unit testing framework, component testers, navigation in tests,
  or mocking Spring beans in Vaadin UI tests.
version: 0.1.0
---

# Browser-Free UI Unit Testing in Vaadin 25

Use the Vaadin MCP tools (`search_vaadin_docs`) to look up the latest documentation whenever uncertain about a specific API detail. Always set `vaadin_version` to `"25"` and `ui_language` to `"java"`.

## What UI Unit Testing Is

UI unit tests run server-side Java code without a browser or servlet container. You interact directly with your server-side view classes and Vaadin components. The `UIUnitTest` base class sets up the Vaadin session, UI, and routing — all in the same JVM as your JUnit tests.

This makes tests fast (milliseconds, not seconds), stable (no browser flakiness), and easy to run in CI.

## When to Use UI Unit Tests vs. End-to-End Tests

**UI unit tests** — the default choice for most view testing:
- Testing view logic, navigation, form validation, component state
- TDD workflows where you run tests on every save
- Verifying that clicking a button shows the right notification
- Testing data binding and Binder behavior

**End-to-end tests (TestBench)** — for critical paths and client-side behavior:
- Login flows, checkout processes
- Testing JavaScript/client-side functionality
- Visual regression testing
- Cross-browser compatibility

Write many UI unit tests and few end-to-end tests.

## Setup

### Maven dependency (Vaadin 25 with JUnit 5/6)

```xml
<dependency>
    <groupId>com.vaadin</groupId>
    <artifactId>vaadin-testbench-unit-junit6</artifactId>
    <scope>test</scope>
</dependency>
```

For JUnit 4, use `vaadin-testbench-unit` and add `junit-vintage-engine`:

```xml
<dependency>
    <groupId>com.vaadin</groupId>
    <artifactId>vaadin-testbench-unit</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.junit.vintage</groupId>
    <artifactId>junit-vintage-engine</artifactId>
    <scope>test</scope>
</dependency>
```

Note: Vaadin 25 with Spring Boot 4 defaults to JUnit 6. JUnit 6 is API-compatible with JUnit 5.

## Writing Your First Test

Extend `UIUnitTest` (JUnit 5/6) or `UIUnit4Test` (JUnit 4):

```java
class HelloWorldViewTest extends UIUnitTest {

    @Test
    void clickButton_showsNotification() {
        // Navigate to the view
        HelloWorldView view = navigate(HelloWorldView.class);

        // Interact via component testers
        test(view.nameField).setValue("Marcus");
        test(view.sayHelloButton).click();

        // Assert on the result
        Notification notification = $(Notification.class).first();
        assertEquals("Hello Marcus", test(notification).getText());
    }
}
```

Key points:
- `navigate(ViewClass.class)` navigates to a route and returns the view instance
- `test(component)` returns a type-specific tester with simulated user actions
- `$(ComponentClass.class)` queries for components in the current UI (like jQuery for Vaadin)
- Fields on the view should be package-private (not private) so tests in the same package can access them

## Navigation

```java
// Simple navigation
MyView view = navigate(MyView.class);

// Navigation with URL parameter
DetailView view = navigate(DetailView.class, "123");

// Navigation with route template parameters
TemplateView view = navigate(TemplateView.class, Map.of("id", "456"));

// Navigation by path string (validates expected view type)
MyView view = navigate("my-view", MyView.class);
```

## Component Testers

The `test()` method returns a tester that simulates user interaction. Testers check that the component is visible, enabled, attached, and not behind a modal before allowing interaction.

```java
// TextField
test(textField).setValue("hello");      // simulates typing
String value = test(textField).getValue();

// Button
test(button).click();                   // simulates click

// Checkbox
test(checkbox).setValue(true);

// ComboBox
test(comboBox).selectItem("Option A");

// Grid — use GridTester for row interaction
GridTester<Person> gridTester = test(grid);
gridTester.clickRow(0);                 // click first row

// Notification
Notification n = $(Notification.class).first();
String text = test(n).getText();
```

For commercial components (Chart, etc.), implement `CommercialTesterWrappers` on your test class.

## Component Queries with $()

Find components in the current UI:

```java
// Find first component of type
Button btn = $(Button.class).first();

// Find by ID
TextField field = $(TextField.class).id("email");

// Find all of a type
List<Button> buttons = $(Button.class).all();

// Check existence
boolean hasGrid = $(Grid.class).exists();

// Nested query — find inside a specific component
VerticalLayout layout = $(VerticalLayout.class).id("content");
Button innerBtn = layout.$(Button.class).first();
```

## Restricting Package Scanning

By default, UIUnitTest scans the entire classpath for routes. For faster startup, restrict to specific packages:

```java
@ViewPackages(classes = {MyView.class, OtherView.class})
class MyViewTest extends UIUnitTest {
    // Only scans packages containing MyView and OtherView
}

// Shortcut — scan only the test class's package
@ViewPackages
class MyViewTest extends UIUnitTest { }
```

## Testing with Spring

For Spring Boot projects, use the Spring-aware test base class:

```java
@SpringBootTest
class MyViewTest extends SpringUIUnitTest {

    @Test
    void testWithSpringContext() {
        MyView view = navigate(MyView.class);
        // Spring beans are injected into the view automatically
    }
}
```

For Quarkus, use `QuarkusUIUnitTest` with `@QuarkusTest`.

## Common Testing Patterns

### Test form validation

```java
@Test
void submitEmptyForm_showsValidationErrors() {
    EditView view = navigate(EditView.class);

    test(view.saveButton).click();

    // Check that required fields show errors
    assertTrue(view.nameField.isInvalid());
}
```

### Test navigation after action

```java
@Test
void saveSuccess_navigatesToList() {
    EditView view = navigate(EditView.class);
    test(view.nameField).setValue("Test");
    test(view.saveButton).click();

    // Verify navigation occurred
    assertTrue(getCurrentView() instanceof ListView);
}
```

### Test dialog content

```java
@Test
void deleteButton_showsConfirmDialog() {
    ListView view = navigate(ListView.class);
    test(view.deleteButton).click();

    ConfirmDialog dialog = $(ConfirmDialog.class).first();
    assertNotNull(dialog);
}
```

## Best Practices

1. **Make view fields package-private** — not private, so tests in the same package can access them directly. This avoids fragile reflection-based lookups.
2. **Use `@ViewPackages` to limit scanning** — speeds up test initialization significantly in large projects.
3. **One assertion focus per test** — test one behavior per method. Name tests descriptively: `action_expectedResult`.
4. **Use `test()` for interaction, not direct method calls** — `test(button).click()` checks visibility and enabled state; `button.click()` bypasses those checks.
5. **Prefer UI unit tests over end-to-end** — they're 10-100x faster and more stable. Reserve end-to-end tests for client-side behavior and critical paths.
6. **Test the user's perspective** — verify what the user sees (notification text, navigation result, field errors) rather than internal implementation details.
