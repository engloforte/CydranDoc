# Node Tree Loading Flow

This document describes how the node tree (nodes + link groups) is loaded, refreshed, and propagated through the app.

## Key Messages (APP_CHANNEL)

- `API_LOAD_TREE` → request to load tree data from the server
- `TREE_LOADED` → broadcast after the server responds with tree data
- `TREE_UPDATED` → broadcast to descendants after AppRoot updates its local state
- `TREE_REQUESTED` → request to rebroadcast the current tree state (no API call)
- `TREE_RELOAD` → request to re-fetch from the server

Defined in: `src/lib/constants.ts`

## Core Components

- `AppRoot` (`src/components/app-root/app-root.ts`)
  - Owns canonical client-side tree state (`nodes`, `linkGroups`, selection).
  - Sends API messages and rebroadcasts tree updates.

- `ApiBroker` (`src/brokers/ApiBroker.ts`)
  - Listens for API messages and calls `ApiService`.
  - Emits `TREE_LOADED` after a successful load.

- `ApiService` (`src/services/api-service.ts`)
  - Performs HTTP requests.

## Initial Load Sequence

1. **AppRoot mounts**
   - Calls `loadTree()`, which sends `API_LOAD_TREE` on `APP_CHANNEL` with `cydran.To.GLOBALLY`.

2. **ApiBroker handles API_LOAD_TREE**
   - Calls `ApiService.loadTree()` (`GET /api/tree`).
   - Emits `TREE_LOADED` globally with `{ nodes, linkGroups, selectedId?, autoFormat? }`.

3. **AppRoot handles TREE_LOADED**
   - Updates local state (`nodes`, `linkGroups`, `selectedId/selectedIds`).
   - Validates selection; falls back to first node if needed.
   - Calls `notifyUpdate()` to send `TREE_UPDATED` to descendants.

4. **Descendants receive TREE_UPDATED**
   - `NodesLayer`, `LinksLayer`, `SelectionOverlay`, editors, etc. refresh their views.

## On-Demand Refresh (No API Call)

- `NodesLayer` (and/or other components) can send `TREE_REQUESTED`.
- `AppRoot` listens to `TREE_REQUESTED` and calls `notifyUpdate()` to rebroadcast the current state via `TREE_UPDATED`.

## Reload from Server

- `TREE_RELOAD` can be sent (e.g., from `TopBar`).
- `AppRoot` handles `TREE_RELOAD` and calls `loadTree()` again (same as initial load).

## API Calls & Follow-Up Loads

All mutations route through `ApiBroker` and then call `loadTree()` to refresh state:

- Create node → `POST /api/nodes` → reload tree (optionally auto-format and select new node)
- Update node → `PATCH /api/nodes/:id` → reload tree
- Delete node → `DELETE /api/nodes/:id` → reload tree
- Add initial → `POST /api/nodes/:id/initials` → reload tree
- Remove initial → `DELETE /api/nodes/:id/initials/:initialId` → reload tree
- Reorder initials → `PATCH /api/nodes/:id/initials` → reload tree
- Add link → `POST /api/links` → reload tree
- Remove link → `DELETE /api/links` → reload tree
- Update link group → `PATCH /api/link-groups/:id` → reload tree
- Reorder children → `PATCH /api/link-groups/:parentId/children` → reload tree
- Apply layout → multiple `PATCH` calls (nodes + link groups) → reload tree

## Where to Look

- Messages/constants: `src/lib/constants.ts`
- API plumbing: `src/brokers/ApiBroker.ts`, `src/services/api-service.ts`
- State + rebroadcast: `src/components/app-root/app-root.ts`
- UI consumers: `src/components/nodes-layer/nodes-layer.ts`, `src/components/links-layer/links-layer.ts`,
  `src/components/selection-overlay/selection-overlay.ts`, editors in `src/components/*`
