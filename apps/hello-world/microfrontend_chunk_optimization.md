# Reducing Angular Material & Microfrontend Chunk Fan-Out

## Problem

Although the application uses a Micro-Frontend (MFE) architecture, the page still takes ~18 seconds to become interactive.

The root issue is not a single slow API, but a chunk fan-out problem caused by:

- Large shared Angular/Material bundles
- Dynamic remote loading
- Global element bootstrapping
- Multiple sequential federation requests
- Runtime environment-based remote resolution

---

# Current Flow

```ts
const [elementsModule, mappingModule] = await Promise.all([
  loadRemoteModule('myblue-core-atoms', './Elements'),
  loadRemoteModule('myblue-core-atoms', './Mapping')
]);

await elementsModule.bootstrapElements();
```

# Recommended Improvements

## 1. Split ./Elements Into Smaller Exposed Modules

### Current

```js
'./Elements': './src/app/elements.ts'
```

### Recommended

```js
'./DrugSearchModal':
  './src/app/elements/drug-search-modal.element.ts',

'./DosageModal':
  './src/app/elements/dosage-modal.element.ts',

'./PharmacyModal':
  './src/app/elements/pharmacy-modal.element.ts',

'./Mapping':
  './src/app/mapping.ts'
```

---

## 2. Load Only Required Components

### Current

```ts
await loadRemoteModule('myblue-core-atoms', './Elements');
```

### Recommended

```ts
await loadRemoteModule(
  'myblue-core-atoms',
  './DrugSearchModal'
);
```

---

## 3. Replace Global Bootstrap

### Current

```ts
await elementsModule.bootstrapElements();
```

### Recommended

```ts
export function defineElement() {
  if (!customElements.get('drug-search-modal')) {
    customElements.define(
      'drug-search-modal',
      DrugSearchModalElement
    );
  }
}
```

---

## 4. Cache Remote Module Loading

```ts
private mappingPromise?: Promise<any>;

private elementPromises =
  new Map<string, Promise<any>>();
```

---

## 5. Avoid shareAll() In Federation

### Bad

```js
shared: {
  ...shareAll({
    singleton: true,
    strictVersion: true,
    requiredVersion: 'auto'
  })
}
```

### Recommended

```js
shared: {
  '@angular/core': {
    singleton: true,
    strictVersion: true,
    requiredVersion: 'auto'
  },

  '@angular/common': {
    singleton: true,
    strictVersion: true,
    requiredVersion: 'auto'
  },

  '@angular/router': {
    singleton: true,
    strictVersion: true,
    requiredVersion: 'auto'
  },

  'rxjs': {
    singleton: true,
    strictVersion: true,
    requiredVersion: 'auto'
  }
}
```

---

## 6. Remove Global Material Module

Avoid:

```ts
imports: [
  MaterialModule
]
```

Use only required Material imports inside standalone components.

---

## 7. Lazy Load Heavy Material Components

Avoid loading these on startup:

- MatTableModule
- MatDatepickerModule
- MatAutocompleteModule
- MatExpansionModule
- MatTabsModule

---

## 8. Preload Only Critical Remotes

Preload only:

- first screen remote
- critical modal remote
- above-the-fold remote

---

# Expected Improvements

| Optimization | Impact |
|---|---|
| Split exposed modules | Huge |
| Remove global bootstrap | Huge |
| Cache remote modules | High |
| Avoid shareAll() | High |
| Remove global Material module | High |
| Lazy load Material | Medium |
| Preload only critical remotes | Medium |

---

# Conclusion

The issue is not Micro-Frontend architecture itself.

The issue is:

- loading too many shared chunks
- loading all custom elements
- large Angular Material dependency graphs
- sequential federation runtime resolution
- excessive shared dependencies
- runtime remote configuration waterfall

The architecture is correct, but the runtime loading strategy needs optimization.
