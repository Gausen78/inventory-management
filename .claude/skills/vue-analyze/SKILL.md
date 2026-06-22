---
name: vue-analyze
description: Analyze Vue 3 component structure and suggest optimizations for performance and code reuse. Use when asked to review, audit, or optimize Vue components in client/src/views/ or client/src/components/.
---

# Vue Component Analyzer

Analyze the specified Vue 3 component(s) for performance issues and code reuse opportunities. If no component is specified, analyze all views in `client/src/views/`.

## How to run

1. **Read** each target `.vue` file in full.
2. **Check** each category below — flag every issue found, not just the first.
3. **Report** findings grouped by category, with file path, line number, severity (High / Medium / Low), and a concrete fix.
4. **Summarize** at the end: total issues by severity, and the top 3 highest-impact changes.

Do **not** apply fixes unless the user explicitly asks. This skill is analysis-only by default.

---

## Performance checks

### Computed vs method misuse
- **Flag**: A `methods` function that derives a value from reactive state and is called in the template more than once, or in a `v-for`.
- **Fix**: Move to a `computed` property so Vue caches the result until dependencies change.
- **Pattern to grep**: `methods:` blocks containing functions used in template interpolation or `:bind` expressions.

### Missing or wrong `v-for` keys
- **Flag**: `v-for` with `:key="index"` or without `:key` at all.
- **Fix**: Use a stable unique ID from the data (e.g., `:key="item.sku"`, `:key="order.id"`).
- **Why**: Index keys cause Vue to reuse the wrong DOM nodes when items are reordered or filtered.

### `v-if` vs `v-show` choice
- **Flag**: `v-if` on an element that toggles frequently (e.g., inside a `watch` or toggled by user interaction multiple times per session).
- **Fix**: Replace with `v-show` when the element is expensive to mount and will be toggled repeatedly.
- **Flag (reverse)**: `v-show` on content that is almost never shown (e.g., error state, empty state shown once).
- **Fix**: Replace with `v-if` to avoid mounting hidden DOM.

### Expensive logic in templates
- **Flag**: Function calls, array methods (`.filter`, `.map`, `.reduce`), or `Object.keys()` directly inside template expressions (not inside a computed).
- **Fix**: Move to a `computed` property. Template expressions run on every render; computed values are cached.

### Unnecessary watchers
- **Flag**: A `watch` that just copies one ref's value into another, or that could be replaced by a `computed`.
- **Flag**: A `watch` with `{ immediate: true }` that also runs the same logic in `onMounted`.
- **Fix**: Consolidate into a single computed or a single `watch` with `immediate: true`.

### API calls not deduplicated
- **Flag**: Multiple watchers or lifecycle hooks that each independently call the same API endpoint for the same data.
- **Fix**: Consolidate into one watcher or share state via a composable.

### Large inline SVG / static data in component
- **Flag**: Hardcoded arrays (e.g., dropdown option lists, color maps, status maps) defined inside `setup()` or `data()` rather than outside the component.
- **Fix**: Move static constants outside the `export default` block so they are not re-created per instance.

---

## Code reuse checks

### Duplicate logic across components
- **Flag**: The same filter construction, date formatting, currency formatting, or status-class mapping appearing in more than one component.
- **Fix**: Extract into a composable in `client/src/composables/` or a shared utility in `client/src/utils/`.
- **Existing composables to check**: look for `useFilters`, `useI18n`, or similar in `client/src/`.

### Component too large
- **Flag**: A `<template>` block over ~150 lines, or a `setup()` function over ~200 lines.
- **Fix**: Identify self-contained sections (e.g., a stats grid, a table, a modal) and extract them as child components in `client/src/components/`.

### Props drilled more than two levels
- **Flag**: A value passed as a prop through an intermediate component that does not use it.
- **Fix**: Use `provide/inject` or promote the value to a shared composable.

### Repeated slot-less wrapper markup
- **Flag**: The same `<div class="card">...<div class="card-header">...<h3 class="card-title">` pattern repeated across multiple views.
- **Fix**: Extract a `Card.vue` or `CardHeader.vue` component.

### API calls duplicated across views
- **Flag**: Two or more views calling the same `api.*` method and storing results in local refs without sharing.
- **Fix**: Consider a shared composable that owns the data and exposes it reactively (similar to how `useFilters` shares filter state).

### Hardcoded strings / magic values
- **Flag**: Status strings (`'Delivered'`, `'Processing'`), warehouse names, or category names hardcoded in component logic (not in data files).
- **Fix**: Move to a shared constants file or derive from the API response keys.

---

## Vue 3 / Composition API correctness

### Options API mixed with Composition API
- **Flag**: `data()`, `computed:`, `methods:` used in the same component as `setup()`.
- **Fix**: Migrate fully to Composition API (`ref`, `computed`, methods defined as functions in `setup`).

### `.value` missing in `<script>`
- **Flag**: A `ref` accessed without `.value` in the script block (a common runtime bug that silently reads the ref wrapper object).
- **Note**: Template access is correct without `.value`; only flag script-side access.

### Prop mutation
- **Flag**: `props.someValue = ...` or direct mutation of a prop object's properties inside the child.
- **Fix**: Emit an event to the parent; never mutate props.

### Date parsing without validation
- **Flag**: `new Date(someString).getMonth()` or `.getFullYear()` without an `isNaN` guard.
- **Fix**: Wrap in `const d = new Date(str); if (!isNaN(d.getTime())) { ... }`.

---

## Report format

For each issue found, output one entry:

```
[SEVERITY] file:line — Category: short description
  Current: <brief code snippet or description>
  Fix: <what to change>
```

Then a final summary:

```
## Summary
- High:   N issues
- Medium: N issues
- Low:    N issues

Top 3 highest-impact changes:
1. ...
2. ...
3. ...
```
