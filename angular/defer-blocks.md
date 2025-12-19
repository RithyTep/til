# Defer Blocks for Lazy Loading

Angular 17+ introduces `@defer` blocks for declarative lazy loading.

## Basic Usage

```html
@defer {
  <heavy-component />
} @placeholder {
  <p>Loading...</p>
}
```

## Trigger Conditions

```html
<!-- Load when visible in viewport -->
@defer (on viewport) {
  <comments-section />
}

<!-- Load on user interaction -->
@defer (on interaction) {
  <rich-editor />
} @placeholder {
  <simple-textarea />
}

<!-- Load after idle time -->
@defer (on idle) {
  <analytics-widget />
}

<!-- Load after delay -->
@defer (on timer(2s)) {
  <promotional-banner />
}
```

## Multiple Conditions

```html
@defer (on viewport; on timer(5s)) {
  <lazy-content />
}
```

## Loading & Error States

```html
@defer {
  <data-table />
} @loading (minimum 500ms) {
  <skeleton-loader />
} @error {
  <error-message />
} @placeholder {
  <p>Content will load when visible</p>
}
```

## Prefetching

```html
@defer (on interaction; prefetch on idle) {
  <modal-dialog />
}
```

## Benefits

- ✅ Automatic code splitting
- ✅ No manual lazy loading setup
- ✅ Declarative in template
- ✅ Built-in loading states
- ✅ Better Core Web Vitals

---

📅 *Learned: 2024*
🏷️ *Tags: Angular, Defer Blocks, Lazy Loading, Performance*
