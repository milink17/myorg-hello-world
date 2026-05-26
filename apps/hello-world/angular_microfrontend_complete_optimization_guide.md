
# Angular Micro-Frontend Performance Optimization Guide

## Executive Summary

Although the application uses a Micro-Frontend (MFE) architecture, the portal still experiences ~18 seconds of startup/loading delay.

The issue is NOT caused by Micro-Frontend architecture itself.

The primary problems are:

- Angular Material chunk fan-out
- Excessive shared federation dependencies
- Runtime remote resolution waterfall
- Loading all elements instead of required ones
- Global custom element bootstrapping
- Sequential remoteEntry loading
- No-cache policies
- Large shared chunks
- Dynamic environment resolution overhead

This document explains:

1. Root Cause Analysis
2. Current Architecture Problems
3. Complete Optimization Strategy
4. Recommended Federation Architecture
5. Angular Material Optimization
6. Runtime Remote Optimization
7. Caching Strategy
8. Deployment Strategy
9. Performance Improvement Expectations
10. Final Recommended Architecture

---

# 1. Root Cause Analysis

## Observed Symptoms

- Portal takes ~18 seconds to become interactive
- Many federation chunk requests
- Large Angular Material shared bundles
- Sequential remote loading
- Runtime configuration waterfall
- Excessive shared dependencies

---

## Actual Runtime Waterfall

```txt
Browser Opens Portal
    ↓
Shell App Loads
    ↓
env.json Requested
    ↓
getenvconfiguration API
    ↓
Resolve Remote URLs
    ↓
Load remoteEntry.js
    ↓
Load Federation Shared Chunks
    ↓
Load Angular Shared Chunks
    ↓
Load Angular Material Chunks
    ↓
Load All Elements
    ↓
bootstrapElements()
    ↓
Register All Components
    ↓
Render Modal
```

This creates excessive network fan-out and startup delay.

---

# 2. Current Architecture Problems

---

## Problem #1 — Single Massive Exposed Module

### Current

```js
'./Elements': './src/app/elements.ts'
```

This likely includes:

- all modals
- all custom elements
- all Angular Material dependencies
- all shared UI atoms
- all CDK dependencies

Result:

Huge dependency graph.

---

## Problem #2 — Global Element Bootstrap

### Current

```ts
await elementsModule.bootstrapElements();
```

This likely:

- registers all custom elements
- loads all dialogs
- loads all atoms
- initializes all Material dependencies

even when only one modal is required.

---

## Problem #3 — Runtime Environment Waterfall

### Current Service

```ts
public async getBaseUrl(id: string): Promise<string> {
  const environment = await getEnvironment();
  const urlConfigs = environment.baseUrls;
  return urlConfigs?.[id]?.[environment.envId] ?? '';
}
```

Problem:

Every remote resolution depends on:

1. environment loading
2. config lookup
3. remote resolution
4. remoteEntry fetch

This creates sequential runtime dependency chains.

---

## Problem #4 — Excessive Shared Federation Chunks

### Current

```js
shared: {
  ...shareAll({
    singleton: true,
    strictVersion: true,
    requiredVersion: 'auto'
  })
}
```

This causes:

- too many shared chunks
- huge shared graph
- Angular Material fan-out
- unnecessary runtime negotiation

---

## Problem #5 — Global MaterialModule

### Current Pattern

```ts
imports: [
  MaterialModule
]
```

This imports ALL Angular Material modules globally.

Result:

- huge shared bundles
- unnecessary startup loading
- excessive federation chunk sharing

---

# 3. Recommended Optimization Strategy

---

# Optimization #1 — Split Exposed Federation Modules

## Current

```js
'./Elements': './src/app/elements.ts'
```

## Recommended

```js
'./DrugSearchModal':
  './src/app/elements/drug-search-modal.element.ts',

'./DosageModal':
  './src/app/elements/dosage-modal.element.ts',

'./PharmacyModal':
  './src/app/elements/pharmacy-modal.element.ts',

'./MemberSearchModal':
  './src/app/elements/member-search-modal.element.ts',

'./Mapping':
  './src/app/mapping.ts'
```

Benefits:

- smaller chunks
- isolated dependency graphs
- lower startup cost
- reduced Angular Material loading

---

# Optimization #2 — Load Only Required Components

## Current

```ts
await loadRemoteModule(
  'myblue-core-atoms',
  './Elements'
);
```

## Recommended

```ts
await loadRemoteModule(
  'myblue-core-atoms',
  './DrugSearchModal'
);
```

Benefits:

- avoids loading unused components
- reduces network requests
- reduces shared chunk negotiation

---

# Optimization #3 — Replace Global Bootstrap

## Current

```ts
await elementsModule.bootstrapElements();
```

---

## Recommended

### Per-element registration

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

Usage:

```ts
const module = await loadRemoteModule(
  'myblue-core-atoms',
  './DrugSearchModal'
);

module.defineElement();
```

Benefits:

- loads only one element
- avoids bootstrapping entire registry
- dramatically reduces startup work

---

# Optimization #4 — Cache Remote Loading

## Current Problem

Every modal open may:

- reload federation resolution
- re-fetch remote
- re-negotiate shared chunks

---

## Recommended

```ts
private mappingPromise?: Promise<any>;

private elementPromises =
  new Map<string, Promise<any>>();
```

### Cache Mapping

```ts
private loadMapping() {

  this.mappingPromise ??=
    loadRemoteModule(
      'myblue-core-atoms',
      './Mapping'
    );

  return this.mappingPromise;
}
```

### Cache Elements

```ts
private loadElement(exposedModule: string) {

  if (!this.elementPromises.has(exposedModule)) {

    this.elementPromises.set(
      exposedModule,
      loadRemoteModule(
        'myblue-core-atoms',
        exposedModule
      )
    );
  }

  return this.elementPromises.get(exposedModule)!;
}
```

Benefits:

- avoids repeated federation negotiation
- avoids repeated network fetches
- faster modal opening

---

# Optimization #5 — Avoid shareAll()

## Current

```js
shared: {
  ...shareAll({
    singleton: true,
    strictVersion: true,
    requiredVersion: 'auto'
  })
}
```

---

## Recommended

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

DO NOT share unless necessary:

- Angular Material
- CDK
- feature libraries
- modal libraries
- utility libraries
- atom libraries

Benefits:

- reduces federation fan-out
- smaller shared graphs
- fewer runtime negotiations

---

# Optimization #6 — Remove Global MaterialModule

## Avoid

```ts
imports: [
  MaterialModule
]
```

---

## Recommended

```ts
imports: [
  MatButtonModule,
  MatDialogModule
]
```

inside standalone components only.

Benefits:

- tree-shaking works properly
- smaller bundles
- fewer federation chunks

---

# Optimization #7 — Lazy Load Heavy Angular Material Components

Avoid loading at startup:

- MatTableModule
- MatDatepickerModule
- MatAutocompleteModule
- MatExpansionModule
- MatTabsModule
- MatTreeModule
- MatStepperModule

Load them only when needed.

---

# Optimization #8 — Preload Only Critical Remotes

After env.json loads:

Preload only:

- first screen remote
- critical modal
- above-the-fold modules

DO NOT preload:

- all atoms
- all dialogs
- all remote elements

---

# Optimization #9 — Enable Long-Term Caching

## Recommended Headers

### For hashed JS bundles

```http
Cache-Control: public, max-age=31536000, immutable
```

### Avoid

```http
Cache-Control: no-cache
Pragma: no-cache
```

on versioned JS bundles.

Benefits:

- browser cache reuse
- avoids repeated federation downloads

---

# Optimization #10 — Reduce Runtime Environment Resolution

## Current

Every remote requires runtime environment lookup.

---

## Recommended

Load environment once during shell bootstrap.

```ts
APP_INITIALIZER
```

Cache all remote URLs in memory.

Example:

```ts
@Injectable({
  providedIn: 'root'
})
export class RemoteConfigService {

  private remoteMap: Record<string, string> = {};

  async initialize() {
    const env = await getEnvironment();

    this.remoteMap = {
      atoms: env.baseUrls.atoms[env.envId],
      dct: env.baseUrls.dct[env.envId]
    };
  }

  getRemoteUrl(name: string) {
    return this.remoteMap[name];
  }
}
```

Benefits:

- avoids repeated async environment lookup
- reduces sequential runtime delays

---

# 4. Recommended Final Architecture

## Recommended Runtime Flow

```txt
Browser Opens Portal
    ↓
Shell Loads
    ↓
APP_INITIALIZER Loads env.json
    ↓
Remote URLs Cached
    ↓
Preload Critical RemoteEntry
    ↓
User Opens Modal
    ↓
Load Specific Exposed Module
    ↓
Register Single Element
    ↓
Render
```

---

# 5. Recommended Federation Architecture

## Shell Responsibilities

- routing
- environment loading
- auth/session
- remote orchestration
- preload critical remotes

---

## Remote Responsibilities

Each remote should own:

- feature UI
- feature Material dependencies
- local feature state
- isolated custom elements

---

## Shared Responsibilities

Only share:

- Angular core
- Angular router
- rxjs

Avoid sharing feature libraries.

---

# 6. Expected Improvements

| Optimization | Expected Impact |
|---|---|
| Split exposed modules | Huge |
| Remove bootstrapElements | Huge |
| Remove shareAll | Huge |
| Remove MaterialModule | Huge |
| Lazy-load Material | High |
| Cache remotes | High |
| Preload critical remotes | Medium |
| Long-term cache headers | High |
| APP_INITIALIZER config preload | High |

---

# 7. Expected Outcome

Expected improvements after optimization:

| Metric | Before | After |
|---|---|---|
| Initial interactive load | ~18s | ~3-5s |
| Modal open time | 3-6s | <1s |
| Federation chunk count | High | Reduced |
| Angular Material startup load | Huge | Minimal |
| Remote negotiation | Repeated | Cached |

---

# 8. Conclusion

The issue is NOT Micro-Frontend architecture.

The issue is runtime loading strategy.

The current architecture behaves like:

"Micro-Frontend Monolith"

because:

- all elements load together
- all Material dependencies load together
- all remotes negotiate shared dependencies
- environment resolution creates waterfalls

The solution is:

- isolate remote modules
- isolate Material dependencies
- remove global bootstrap
- cache remote loading
- avoid shareAll()
- preload selectively
- optimize runtime environment loading

Once optimized, the architecture will achieve:

- true independent micro-frontends
- faster startup
- minimal federation overhead
- isolated feature delivery
- scalable runtime performance
