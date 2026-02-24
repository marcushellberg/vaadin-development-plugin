---
name: data-providers
description: >
  Guide Claude on using Vaadin 25 data providers efficiently for Grid, ComboBox,
  and other listing components. This skill should be used when the user asks to
  "load data into a grid", "use a data provider", "lazy load data",
  "paginate grid data", "filter a grid", "sort a grid", "use setItems",
  "use CallbackDataProvider", "bind a Spring service to a grid",
  "use setItemsPageable", or needs help with in-memory vs lazy data binding,
  filtering, sorting, or custom data providers in Vaadin Flow.
version: 0.1.0
---

# Vaadin Data Providers in Vaadin 25

Use the Vaadin MCP tools (`search_vaadin_docs`, `get_component_java_api`) to look up the latest documentation whenever uncertain about a specific API detail. Always set `vaadin_version` to `"25"` and `ui_language` to `"java"`.

## Two Approaches: In-Memory vs. Lazy Loading

### In-memory — small datasets that fit in server memory

```java
List<Person> people = personService.findAll();
grid.setItems(people);
```

Use when: the full dataset is small (hundreds to low thousands of items), or you've already loaded it for other reasons.

In-memory data supports client-side-like sorting and filtering through the `ListDataView`:

```java
GridListDataView<Person> dataView = grid.setItems(people);
dataView.setFilter(p -> p.getAge() < 30);
dataView.setSortOrder(Person::getName, SortDirection.ASCENDING);
```

### Lazy loading — large datasets loaded on demand

Only the visible portion of data is loaded. Filtering and sorting are delegated to the backend (typically a database).

**With Spring (preferred — uses `Pageable`):**

```java
grid.setItemsPageable(personService::list);

// Service method signature:
public List<Person> list(Pageable pageable) {
    return repository.findAll(pageable).getContent();
}
```

`setItemsPageable` converts Grid's internal query into a Spring `Pageable` with offset, limit, and sort information. This integrates directly with Spring Data repositories.

**Without Spring (generic callbacks):**

```java
grid.setItems(query ->
    personService.fetch(query.getOffset(), query.getLimit()).stream()
);
```

The callback receives a `Query` object with `getOffset()`, `getLimit()`, and `getSortOrders()`.

## Filtering

### In-memory filtering

```java
GridListDataView<Person> dataView = grid.setItems(allPeople);

filterField.addValueChangeListener(e ->
    dataView.setFilter(person ->
        person.getName().toLowerCase().contains(e.getValue().toLowerCase()))
);
```

### Lazy filtering (backend-delegated)

Pass the filter value into your service callback:

```java
// With Spring
GridLazyDataView<Person> dataView = grid.setItemsPageable(
    pageable -> personService.list(pageable, filterField.getValue())
);
filterField.setValueChangeMode(ValueChangeMode.LAZY);
filterField.addValueChangeListener(e -> dataView.refreshAll());
```

```java
// Without Spring
grid.setItems(query ->
    personService.fetch(
        query.getOffset(),
        query.getLimit(),
        query.getSortOrders(),
        filterField.getValue()
    ).stream()
);
filterField.addValueChangeListener(e -> grid.getDataProvider().refreshAll());
```

Use `ValueChangeMode.LAZY` on the filter field — it waits for the user to pause typing before triggering, reducing backend calls.

## Sorting

### In-memory sorting

Columns with `Comparable` types are automatically sortable. Customize with a `Comparator`:

```java
grid.addColumn(Person::getName)
    .setHeader("Name")
    .setComparator(Comparator.comparing(p -> p.getName().toLowerCase()));
```

### Lazy sorting (backend-delegated)

Declare which columns are sortable:

```java
grid.setSortableColumns("name", "email");  // by property name
// or per-column:
grid.addColumn(Person::getTitle).setKey("title").setSortable(true);
```

Then handle `query.getSortOrders()` in your callback. With `setItemsPageable`, sorting is handled automatically via the `Pageable` object.

## ComboBox Lazy Loading

ComboBox supports lazy loading with built-in text filtering:

```java
// With Spring
comboBox.setItemsPageable(productService::list);

// Service must accept filter string:
public List<Product> list(Pageable pageable, String filterString) { ... }
```

```java
// Without Spring
comboBox.setItems(query ->
    personService.fetch(
        query.getOffset(),
        query.getLimit(),
        query.getFilter().orElse("")
    ).stream()
);
```

## Providing Item Count

Without a count callback, Grid uses infinite scrolling (the scrollbar updates as more items are loaded). For a better UX, provide the count:

```java
// With Spring
grid.setItemsPageable(personService::list, personService::count);

// Without Spring
grid.setItems(
    query -> personService.fetch(query.getOffset(), query.getLimit()).stream(),
    query -> personService.count()
);
```

If exact count is expensive, provide an estimate:

```java
GridLazyDataView<Person> dataView = grid.setItems(fetchCallback);
dataView.setItemCountEstimate(1000);
dataView.setItemCountEstimateIncrease(500);
```

## Updating Displayed Data

### Refresh a single item

```java
person.setEmail("new@example.com");
dataView.refreshItem(person);  // updates just this row
```

### Refresh all data

```java
dataView.refreshAll();  // re-fetches everything
```

### Mutating in-memory lists

```java
GridListDataView<Person> dataView = grid.setItems(mutableList);
dataView.addItem(newPerson);
dataView.removeItem(oldPerson);
```

## Item Identity

Components rely on `hashCode()` and `equals()` for item identity. These **must** be based on stable, unique properties (typically the database ID), not mutable fields:

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    Person person = (Person) o;
    return id == person.id;  // stable identifier only
}

@Override
public int hashCode() {
    return Objects.hash(id);
}
```

With Lombok: use `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` and `@EqualsAndHashCode.Include` on the ID field.

## Reusing Data Binding Logic

### Domain-specific Grid subclass

```java
@SpringComponent
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class PersonGrid extends Grid<Person> {

    public PersonGrid(@Autowired PersonRepository repo) {
        super(Person.class);
        setItems(q -> repo.findAll(
            PageRequest.of(q.getPage(), q.getPageSize())).stream());
        setColumns("name", "email");
    }
}
```

### Standalone DataProvider class

```java
@SpringComponent
public class PersonDataProvider
        extends AbstractBackEndDataProvider<Person, String> {

    @Autowired
    private PersonRepository repo;

    @Override
    protected Stream<Person> fetchFromBackEnd(Query<Person, String> query) {
        String filter = query.getFilter().orElse("");
        return repo.findByNameContaining(filter,
            VaadinSpringDataHelpers.toSpringPageRequest(query)).stream();
    }

    @Override
    protected int sizeInBackEnd(Query<Person, String> query) {
        String filter = query.getFilter().orElse("");
        return (int) repo.countByNameContaining(filter);
    }
}
```

## Best Practices

1. **Use `setItemsPageable` with Spring** — it handles offset/limit/sort conversion to `Pageable` automatically, integrating cleanly with Spring Data.
2. **Use lazy loading for datasets over ~1000 items** — in-memory is simpler but doesn't scale.
3. **Always implement stable `hashCode`/`equals`** — based on the entity's ID, not mutable fields. This is required for Grid to correctly track selections and updates.
4. **Use `ValueChangeMode.LAZY` on filter fields** — prevents excessive backend queries while the user is typing.
5. **Provide a count callback when possible** — it gives users a proper scrollbar and ability to jump to the end.
6. **Call `refreshAll()` when filter/sort criteria change externally** — the Grid doesn't automatically know when your filter field value changes.
7. **Don't call `getGenericDataView().getItems()` on lazy data** — it loads the entire dataset into memory, defeating the purpose of lazy loading.
