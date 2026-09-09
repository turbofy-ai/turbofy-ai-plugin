# Shared utilities and state

Use app-scoped shared modules for code reused by multiple blocks: formatting, types, components, data transformations, and client state. Keep block-specific code beside that block.

## Files and imports

In a hosted app tree, create one directory per shared module:

```text
shared/
  Formatting/
    index.ts
    format-number.ts
  Selection/
    index.ts
```

A module needs `index.ts` or `index.tsx`. It has no `record.ts` and does not belong in `buildApp.blockTypes` or the block-type barrel. Use names containing letters, digits, dots, dashes, or underscores.

Import its public exports from `@/shared/<Name>` in any block or another shared module:

```ts
import { formatNumber } from "@/shared/Formatting";
import { useSelection } from "@/shared/Selection";
```

Re-export helpers from the module's index. Import siblings relatively within the module; do not deep-import `@/shared/Formatting/format-number` or reach into another block's folder. Avoid circular dependencies between shared modules.

## A store shared by blocks

Declare the store once at module scope, rather than creating a store in each block. For example, add `zustand` to the app's `package.json` if needed, then write `shared/Selection/index.ts`:

```ts
import { create } from "zustand";

interface SelectionState {
  selectedId: string | null;
  select: (id: string | null) => void;
}

export const useSelection = create<SelectionState>((set) => ({
  selectedId: null,
  select: (selectedId) => set({ selectedId }),
}));
```

A list block can call `useSelection((state) => state.select)`; a details block can read `useSelection((state) => state.selectedId)`. Both import the same module, so updates are visible across those blocks. Use selectors to subscribe to only the state a block needs.

This is client UI state, not durable storage or synchronization between visitors, tabs, or devices. Store durable data through Turbofy data APIs. Keep browser-only work out of module initialization when the module is also used by server-rendered blocks. Use `@/lib/auth` for session state.

## Hosted workflow

1. `app_pull`, then inspect existing `shared/` modules before adding another.
2. Edit with `fs_*`; add dependencies to the app's `package.json` and install them with `fs_exec` when needed.
3. Check consuming blocks with `block_type_check` and run `app_push` with `dryRun: true`. Inspect `sharedModules` and block failures as well as TypeScript results.
4. Apply with `dryRun: false`, inspect the reported failures, and verify the consuming blocks together in preview.

`app_push` includes shared modules automatically. Removing a previously pulled module directory requests deletion; renaming it requires updating all imports. A module added remotely after your pull is kept until you pull it into your tree.
