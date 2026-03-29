# System Patterns: Excalidraw

**Version:** 1.0
**Date:** 2026-03-28
**Status:** Active

---

## 1. Introduction

This document captures the recurring design patterns, architectural decisions, and technical conventions used across the Excalidraw codebase. It serves as the canonical reference for:

- **Consistency** — ensuring new features align with established approaches rather than introducing ad-hoc alternatives.
- **Onboarding** — helping new engineers and contributors understand *how* the system is built, not just *what* it does.
- **Future scalability** — providing a baseline against which architectural evolution can be measured and reasoned about.

Audience: engineers, architects, and tech leads working on or integrating with the Excalidraw monorepo.

---

## 2. Architectural Style

Excalidraw is a **client-side monolithic SPA** with a thin layer of external backend integrations. It is not microservices; there is no application server in this repository.

| Dimension              | Description                                                                                                             |
|------------------------|-------------------------------------------------------------------------------------------------------------------------|
| **Rendering model**    | Single-page application (SPA) — no SSR; Vite-bundled React                                                              |
| **Deployment targets** | Web app (`excalidraw-app/`) + embeddable npm component (`packages/excalidraw/`)                                         |
| **Package topology**   | Yarn workspaces monorepo: `common → math → element → excalidraw → excalidraw-app`                                       |
| **External backends**  | Firebase (persistence), Socket.IO server (collab relay), AI API (TTD), Sentry (errors) — none of these are in this repo |
| **Offline capability** | Full offline-first; localStorage + IndexedDB ensure zero-network operation                                              |
| **Real-time**          | Collaborative sessions use a relayed WebSocket model (not peer-to-peer)                                                 |

The architecture intentionally keeps all business logic, encryption, and reconciliation on the **client**. Servers act as dumb relays or opaque storage — they never see plaintext diagram data.

---

## 3. Core Patterns

### 3.1 State Management — Hybrid Jotai + React State

Excalidraw uses a **two-tier state model**:

| Tier                      | Technology       | Scope                                      | Location                                                             |
|---------------------------|------------------|--------------------------------------------|----------------------------------------------------------------------|
| **Global UI / app state** | Jotai atoms      | Cross-component, persistent within session | `excalidraw-app/app-jotai.ts`, `packages/excalidraw/editor-jotai.ts` |
| **Component-local state** | `React.useState` | Ephemeral, UI-only                         | Per-component                                                        |

**Why Jotai?** Selected over Redux for its minimal boilerplate, atomic granularity (avoids over-rendering), and compatibility with React concurrent features. `jotai-scope` is used in `editor-jotai.ts` to isolate multiple `<Excalidraw />` instances on the same page, which is critical for the embeddable component.

**AppState vs Element State separation:**

- `AppState` — transient UI state (tool selection, zoom, scroll, open panels). Stored in Jotai atoms and selectively persisted to localStorage via `clearAppStateForLocalStorage()`.
- `Element state` — the canonical scene data. Managed by the `Store` class (`packages/element/src/store.ts`) as an immutable snapshot; never stored directly in React state.

**Anti-pattern avoided:** No Redux, no MobX, no Context-for-data (Context is used only for providing API handles, not for reactive data flow).

---

### 3.2 Scene Store — Immutable Snapshot + Increment Model

**Files:** `packages/element/src/store.ts`, `packages/common/src/versionedSnapshotStore.ts`

The scene is managed through an **immutable snapshot + named increment** pattern:

```
User interaction → App.mutateElement() → Store.capture()
                                          ├─ DurableIncrement  → undo/redo history + collab broadcast
                                          └─ EphemeralIncrement → collab cursor/idle (not stored)
```

- `StoreSnapshot` — immutable point-in-time copy of all elements and appState.
- `DurableIncrement` — a named, versioned change with a `StoreDelta` (inserted/deleted element sets). Feeds into undo/redo (`History`) and collaboration delta sync.
- `EphemeralIncrement` — transient change (cursor movement, idle status). Broadcast but not stored.
- `CaptureUpdateAction` — enum (`IMMEDIATELY` / `EVENTUALLY` / `NEVER`) that controls whether a mutation creates a durable history entry.

`VersionedSnapshotStore` (generic) provides versioned state with a `pull()` API — callers can await the next version, enabling cross-tab synchronisation without polling.

---

### 3.3 Action Pattern — Centralised Command Registry

**Files:** `packages/excalidraw/actions/`

All user-level operations are implemented as **registered actions** — pure functions with a standard signature:

```typescript
const myAction = register({
  name: "myAction",
  perform: (elements, appState, value, app): ActionResult => {
    return {
      elements: [...],
      appState: { ...appState, ... },
      captureUpdate: CaptureUpdateAction.IMMEDIATELY,
    };
  },
});
```

- **`register()`** appends the action to a global registry. The `ActionManager` discovers all registered actions at startup.
- Actions are pure: they receive current state and return a new state — no side effects inside `perform`.
- `ActionSource` (`"ui"` / `"keyboard"` / `"api"` / `"commandPalette"`) is tracked for analytics.
- The Command Palette consumes the same registry, giving all actions keyboard-searchable access for free.

This pattern enforces a single, auditable path for every state-modifying operation.

---

### 3.4 Event Bus / Emitter Pattern

**Files:** `packages/common/src/emitter.ts`, `packages/common/src/appEventBus.ts`

Two levels of event infrastructure are used:

**`Emitter<T>` (base):** Lightweight typed pub/sub. Used internally by the `Store` to emit `onDurableIncrementEmitter` / `onStoreIncrementEmitter` events to the `History` and `Collab` layers.

**`AppEventBus` (application-level):** A richer event bus with:
- **Cardinality control** — `"once"` (fires at most once) or `"many"`.
- **Replay behaviour** — `"last"` replays the most recent payload to late subscribers; `"none"` does not.
- **`await`-able events** — callers can `await appEventBus.on('eventName')` for one-time lifecycle events (e.g., app initialised).

Both buses are strictly typed — event names and payload shapes are defined in TypeScript, preventing untyped string-based event chains.

---

### 3.5 Real-Time Collaboration — Encrypted Relay Model

Clients communicate through an encrypted Socket.IO relay (never peer-to-peer). `Portal` sends only changed elements (delta sync) with a periodic 20-second full-scene broadcast as a safety net. Every payload is AES-GCM encrypted before leaving the client. The `Collab` class orchestrates Portal, FileManager, LocalData, and Firebase.

See [`docs/technical/realtime-collaboration.md`](../technical/realtime-collaboration.md) for protocol constants, class responsibilities, and sync timing details.

---

### 3.6 End-to-End Encryption

All collaboration and share-link data is AES-GCM encrypted client-side (via `window.crypto.subtle`) before leaving the browser. The encryption key lives only in the URL fragment and is never transmitted to any server. Firebase and the Socket.IO relay store and forward only ciphertext.

See [`docs/technical/encryption.md`](../technical/encryption.md) for algorithm parameters, key distribution model, and known trade-offs.

---

### 3.7 Conflict Reconciliation — Last-Write-Wins with Nonce Tiebreaker

Concurrent edits are resolved with a Last-Write-Wins strategy using `version` (monotonic) and `versionNonce` (random tiebreaker) per element. Locally-active elements are protected from remote overwrites. After merging, fractional indices are repaired to preserve z-order. This is not OT or CRDT — eventual consistency via periodic full-scene syncs is sufficient for a whiteboard.

See [`docs/technical/conflict-reconciliation.md`](../technical/conflict-reconciliation.md) for the full decision algorithm and known limitations.

---

### 3.8 Data Persistence — Layered Offline-First Strategy

Persistence uses four layers: localStorage (L1) → IndexedDB (L2) → Firebase Firestore (L3) → Firebase Storage (L4). On load, the app hydrates from the highest available layer. Firebase is authoritative for collab sessions; localStorage for solo. Cross-tab sync uses `BroadcastChannel` backed by `VersionedSnapshotStore`.

See [`docs/technical/data-persistence.md`](../technical/data-persistence.md) for layer capacities, write triggers, hydration rules, and AppState filtering details.

---

### 3.12 Error Handling

A React Error Boundary (`TopErrorBoundary`) catches unhandled render errors, captures them in Sentry, and presents a fallback UI. Sentry is enabled only in production/staging and sanitises URLs before sending (stripping the encryption key fragment). Network/storage failures degrade gracefully to offline mode — no circuit breakers or retries.

See [`docs/technical/error-handling.md`](../technical/error-handling.md) for full implementation details.

---

## 4. Patterns to Avoid

| Anti-Pattern                                 | Reason Avoided                                                                                                                          |
|----------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| **Redux / global mutable store**             | Over-engineered for a client-side SPA with well-scoped state; Jotai provides fine-grained reactivity without action/reducer boilerplate |
| **Server-side rendering (SSR)**              | Canvas API and Web Crypto API are browser-only; SSR would require polyfills and provide no meaningful benefit for a drawing tool        |
| **Transmitting encryption keys to servers**  | Zero-trust principle; keys live only in URL fragments and are never sent to Firebase, Socket.IO relay, or any logging pipeline          |
| **Storing plaintext scene data server-side** | All server/cloud storage holds only AES-GCM ciphertext; servers are treated as untrusted relays                                         |
| **Higher-Order Components (HOCs)**           | React hooks and compound components provide the same composition without the wrapper-hell debugging issues                              |
| **Direct DOM mutation outside React**        | Canvas drawing goes through `requestAnimationFrame` + Canvas 2D API (not DOM); all React-managed UI is mutated through state only       |
| **Polling for real-time updates**            | WebSocket (Socket.IO) for collab, BroadcastChannel for cross-tab sync, and `VersionedSnapshotStore.pull()` replace all polling patterns |
| **Synchronous encryption**                   | All crypto operations use the async `window.crypto.subtle` API — synchronous crypto would block the main thread                         |
| **Bundling all locales upfront**             | Locale files are lazy-loaded to avoid bloating the initial bundle; only the active locale is fetched                                    |

---

## 5. Related Documentation

| Document | Location | Relationship |
|---|---|---|
| **Technical Architecture** | [`docs/technical/architecture.md`](../technical/architecture.md) | Full system design document; the patterns described here are grounded in the architectural decisions and component topology detailed there |
| **Product Requirements Document** | [`docs/product/PRD.md`](../product/PRD.md) | Product requirements that shaped specific architectural choices — e.g., the offline-first storage pattern, E2E encryption design, and embeddable component API surface |
| **Error Handling** | [`docs/technical/error-handling.md`](../technical/error-handling.md) | Full implementation detail for the error boundary, Sentry integration, and graceful degradation strategy introduced in §3.12 |
| **Real-Time Collaboration** | [`docs/technical/realtime-collaboration.md`](../technical/realtime-collaboration.md) | Protocol constants, class responsibilities, and sync timing for the encrypted relay model introduced in §3.5 |
| **End-to-End Encryption** | [`docs/technical/encryption.md`](../technical/encryption.md) | AES-GCM algorithm parameters, key distribution model, and trade-offs introduced in §3.6 |
| **Conflict Reconciliation** | [`docs/technical/conflict-reconciliation.md`](../technical/conflict-reconciliation.md) | LWW algorithm, tiebreaker logic, and z-order repair introduced in §3.7 |
| **Data Persistence** | [`docs/technical/data-persistence.md`](../technical/data-persistence.md) | Storage layer details, write triggers, hydration order, and cross-tab sync introduced in §3.8 |
