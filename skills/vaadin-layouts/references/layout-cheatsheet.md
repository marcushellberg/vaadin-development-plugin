# Layout Quick-Reference Cheatsheet

## Default Comparison

| Property        | VerticalLayout | HorizontalLayout |
|-----------------|----------------|------------------|
| Width           | 100% (fills)   | undefined (hugs) |
| Height          | undefined (hugs) | undefined (hugs) |
| Spacing         | ON             | ON               |
| Padding         | ON             | OFF              |
| Margin          | OFF            | OFF              |
| Align items     | START (left)   | STRETCH (vertical) |
| Justify content | START (top)    | START (left)     |

## Alignment Values

### JustifyContentMode (main axis distribution)

| Value     | Effect |
|-----------|--------|
| `START`   | Pack to start (default) |
| `CENTER`  | Center all items |
| `END`     | Pack to end |
| `BETWEEN` | Even space between, none at edges |
| `AROUND`  | Even space around, half at edges |
| `EVENLY`  | Perfectly even space everywhere |

### Alignment (cross axis)

| Value     | Effect |
|-----------|--------|
| `START`   | Align to start edge |
| `CENTER`  | Center on cross axis |
| `END`     | Align to end edge |
| `STRETCH` | Fill cross axis (if size undefined) |
| `BASELINE`| Align text baselines (HorizontalLayout only) |

## Spacing Theme Variants

To use a non-default spacing size:

```java
layout.setSpacing(false);                    // disable default
layout.getThemeList().add("spacing-xs");     // add variant
```

| Variant       | Size |
|--------------|------|
| `spacing-xs` | Extra small |
| `spacing-s`  | Small |
| `spacing`    | Medium (default when enabled) |
| `spacing-l`  | Large |
| `spacing-xl` | Extra large |

Or use custom pixel values:

```java
layout.setSpacing(8, Unit.PIXELS);
```

## CSS Custom Properties

| Property | Component |
|----------|-----------|
| `--vaadin-horizontal-layout-gap` | HorizontalLayout |
| `--vaadin-horizontal-layout-padding` | HorizontalLayout |
| `--vaadin-horizontal-layout-margin` | HorizontalLayout |
| `--vaadin-vertical-layout-gap` | VerticalLayout |
| `--vaadin-vertical-layout-padding` | VerticalLayout |
| `--vaadin-vertical-layout-margin` | VerticalLayout |

## Common Patterns

### Toolbar with left and right sections

```java
HorizontalLayout toolbar = new HorizontalLayout();
toolbar.setWidthFull();
toolbar.setPadding(true);
toolbar.setJustifyContentMode(FlexComponent.JustifyContentMode.BETWEEN);
toolbar.setAlignItems(FlexComponent.Alignment.CENTER);

toolbar.add(leftGroup, rightGroup);
```

### Centered card in full-height view

```java
VerticalLayout view = new VerticalLayout();
view.setSizeFull();
view.setJustifyContentMode(FlexComponent.JustifyContentMode.CENTER);
view.setAlignItems(FlexComponent.Alignment.CENTER);

view.add(card);
```

### Sidebar + content area

```java
HorizontalLayout shell = new HorizontalLayout();
shell.setSizeFull();
shell.setSpacing(false);
shell.setPadding(false);

VerticalLayout sidebar = new VerticalLayout();
sidebar.setWidth("250px");
shell.setFlexShrink(0, sidebar); // prevent sidebar from shrinking

VerticalLayout content = new VerticalLayout();
shell.setFlexGrow(1, content);   // content takes remaining space

shell.add(sidebar, content);
```

### Scrollable content area

```java
VerticalLayout content = new VerticalLayout();
content.setMinHeight("0"); // allow shrinking below content size
// Wrap in a Scroller, or set overflow on a parent
```

## Choosing the Right Layout Component

| Need | Use | NOT |
|------|-----|-----|
| App shell with nav drawer + header + content | `AppLayout` | Nested HorizontalLayout/VerticalLayout |
| Grid of cards / dashboard widgets | `Dashboard` or CSS Grid | Wrapped HorizontalLayouts |
| Form field arrangement | `FormLayout` | Manual VerticalLayout with spacing |
| Single row of components | `HorizontalLayout` | — |
| Single column of components | `VerticalLayout` | — |
| Dynamic horizontal/vertical toggle | `FlexLayout` | Swapping between H/V Layout |

## Vaadin 25 Sizing Behavior

In Vaadin 25, `setWidthFull()` / `setHeightFull()` / `setSizeFull()` set `flex: 1 1 100%` (not a literal `width: 100%`). This means they take available space while respecting siblings. Fixed-size siblings will NOT shrink unexpectedly.

> **Legacy note (Vaadin 24 and earlier):** In older versions, these methods set a literal `width: 100%` / `height: 100%`, which caused siblings to shrink. Old advice about always using `setFlexGrow` instead of `setWidthFull` is outdated for Vaadin 25.

## Troubleshooting Decision Tree

**Problem: Component renders smaller than specified size**
1. Is the parent constraining it? → Check parent sizing chain up to the root
2. Is `setFlexShrink` set to a non-zero value? → Set `setFlexShrink(0, component)` to prevent shrinking

**Problem: Unwanted scrollbars / overflow**
1. Is it a nested layout? → Set `setMinHeight("0")` or `setMinWidth("0")` on the overflowing component
2. Is content just too wide? → Enable wrapping with `setWrap(true)` on HorizontalLayout

**Problem: Extra whitespace around content**
1. Are you nesting layouts? → Disable padding/spacing on inner layouts: `setPadding(false)`, `setSpacing(false)`
2. Is margin enabled? → Check `setMargin(false)`
