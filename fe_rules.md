author: Daniel Zikmund, zikmund.d@gmail.com
version: 1.0.0

# Frontend Architecture Rules

Reusable rules for any React frontend built on **Feature Sliced Design**. These are _non-negotiable_ unless explicitly overridden in a project's own spec.

## 1. Stack baseline

- **Bun** as package manager + runtime (lockfile checked in)
- **Vite** as dev server / bundler
- **React 19 + TypeScript** with `strict: true`
- **Tailwind CSS + shadcn/ui** for styling and UI primitives
- **React Router** for routing
- **TanStack Query** for server state — never hand-rolled `useEffect(fetch)`
- **React Hook Form + Zod** for forms and validation — same Zod schemas reused for runtime + compile-time types
- **Vitest + React Testing Library** for tests; **MSW** for network mocking

## 2. Feature Sliced Design layers

From top (composition) to bottom (primitives):

```
app/        →  providers, router, entry point
pages/      →  one module per route
widgets/    →  composite UI blocks reused across pages
features/   →  user-facing interactions that change state
entities/   →  domain nouns (display primitives + queries)
shared/     →  UI kit, API transport, config, utils, types
```

### Layer import direction

A layer may only import from layers **below** it. Never sideways or upward.

- `app` → everything below
- `pages` → widgets, features, entities, shared
- `widgets` → features, entities, shared
- `features` → entities, shared
- `entities` → shared
- `shared` → nothing (leaf layer)

**Enforce with `eslint-plugin-boundaries` or a custom ESLint rule. A violation is a build failure.**

## 3. Slice rules

Every slice (`entities/pipeline`, `features/pipelines/run-pipeline`, etc.) has this structure:

```
<slice>/
├── ui/          # React components (only ones exported for others to use)
├── model/       # types, Zod schemas, local state, helpers
├── api/         # TanStack Query hooks
├── lib/         # slice-internal utilities (optional)
├── config/      # slice-internal constants (optional)
└── index.ts     # PUBLIC API — the ONLY importable entry point
```

- **Public API via `index.ts`.** Outside code imports `from "entities/pipeline"` — never `from "entities/pipeline/ui/PipelineRow"`.
- Deep imports across slices are forbidden (ESLint rule).
- A slice owns everything it needs. No "utils.ts" at the layer root that everyone dips into — that's a smell, push into `shared/lib`.

## 4. What belongs in which layer

### `shared/`

- UI kit (shadcn components, design system primitives)
- API transport (`apiClient`, error types, query client config)
- Config (env, routes, query keys)
- Pure utility functions and hooks (`formatDate`, `useDebounce`)
- Generated API types (e.g. from OpenAPI)
- **No domain concepts here.** If it mentions a business noun, it's not shared.

### `entities/`

- Domain nouns as UI + state: `entities/dataset`, `entities/pipeline`
- Read queries (`useXxxQuery`, `useXxxByIdQuery`)
- Display components (`PipelineRow`, `StatusBadge`)
- Zod schemas matching the domain model
- **No mutations.** Mutations are user actions → features.

### `features/`

- Single user interaction: `features/pipelines/run-pipeline`
- Contains the trigger (button/form), the mutation hook, and any dialog/form
- Named by action: verbs, not nouns (`create-dataset`, not `dataset-creator`)
- A feature is deletable — removing it must not break anything outside the feature itself

### `widgets/`

- Composite blocks reused across pages (`PipelineTable`, `DashboardSummaryCards`)
- Wire entities + features together into a self-contained block
- Own the loading / empty / error states for the data they render

### `pages/`

- One folder per route. Thin composition only.
- Page reads URL params, composes widgets + features, nothing more
- No business logic, no direct API calls

### `app/`

- Providers: `QueryClientProvider`, `AuthProvider`, `ThemeProvider`, `RouterProvider`
- Global styles entry point
- Single `main.tsx` entry point
- Route table

## 5. Loading / empty / error states

Every query-backed UI renders three distinct views. This is a **hard requirement**, not a nice-to-have.

- **Loading** — shadcn `Skeleton` matching the final layout, not spinners
- **Empty** — `<EmptyState>` component with a clear CTA (when action is possible)
- **Error** — `<ErrorState>` component showing the problem and a retry button

`EmptyState` and `ErrorState` components live in `shared/ui/` and are used everywhere.

## 6. API integration

- Generate types from the backend's OpenAPI spec into `shared/types/api.generated.ts`. Never hand-write types that mirror the API.
- `apiClient` is the single `fetch` wrapper. It injects auth, parses JSON, throws typed `ApiError`.
- Every mutation that modifies server state invalidates the relevant queries on success (`queryClient.invalidateQueries(['pipelines'])`). Inconsistent cache = bug.
- Query keys follow a hierarchy: `['pipelines']`, `['pipelines', id]`, `['pipelines', id, 'runs']`. Constants defined in `shared/config/queryKeys.ts`.

## 7. State management

- **Server state** → TanStack Query. Period.
- **URL state** (filters, pagination, active tab) → `useSearchParams`. Shareable URLs matter.
- **Local component state** → `useState` / `useReducer`.
- **Cross-component UI state** → Zustand, only when URL + server state are not enough.
- **Global mutable state** is a last resort. If you reach for it, first ask whether it's actually server state that wants invalidation.

## 8. Forms

- React Hook Form + Zod. Never manage form state by hand.
- Zod schema is shared between form validation and API contract types.
- Submit handlers call the mutation hook from the feature and show toasts on success/error.
- Field components wrap shadcn's `Form` primitives — no ad-hoc `<input>` tags.

## 9. Routing

- Single route table in `app/router/`.
- Route paths defined as constants in `shared/config/routes.ts`. Build URLs via `ROUTES.pipelines.detail(id)` — never string concatenation.
- Protected routes wrap pages in a `ProtectedRoute` component. Role-gated UI uses a `<RoleGuard>` component from `entities/user`.

## 10. Styling

- Tailwind utility classes for layout and spacing.
- shadcn components for anything interactive. Do not reinvent a `Dialog` when `shadcn/ui Dialog` exists.
- No CSS modules. No styled-components. No plain `.css` files outside `shared/ui`.
- Design tokens (colors, spacing) live in `tailwind.config.ts`. Components reference them by name, never hex codes inline.

## 11. Documentation (JSDoc)

- Every exported function, hook, and component has a JSDoc block with a one-sentence summary and `@param` / `@returns` where applicable.
- Document the _why_, not the _what_. A component's JSDoc explains when to use it, not what's inside.
- Complex types exported from a slice carry JSDoc describing each field.
- Internal helpers document only the non-obvious — skip when the name says everything.

## 12. TypeScript

- `strict: true` with `noUncheckedIndexedAccess: true`.
- No `any`. If you must use it, add an inline comment explaining why and plan to remove it.
- Prefer `interface` for object shapes, `type` for unions and utilities.
- Discriminated unions over boolean flags (`{ status: 'loading' } | { status: 'error', error } | { status: 'success', data }`).

## 13. Testing

- Colocate tests: `Component.tsx` + `Component.test.tsx` in the same folder.
- Unit test utilities, hooks, model helpers. Component test widgets and features (user behavior, not implementation).
- Mock network with MSW — hitting the real apiClient proves the wiring works.
- Smoke E2E (Playwright) for the critical happy paths (login, the one action the app exists for).

## 14. Environment variables

- Prefix with `VITE_`.
- Validated at app boot via a Zod schema in `shared/config/env.ts`. Startup fails fast on missing/malformed vars.
- Never read `import.meta.env` outside `shared/config/env.ts`.

## 15. Naming

- Files: `PascalCase.tsx` for components, `camelCase.ts` for everything else.
- Folders: `kebab-case`.
- Hooks: `useXxx`, always starting with `use`.
- Components: `PascalCase`, named the same as their file.
- Event handlers: `handleXxx` inside a component, `onXxx` on props.
- Booleans: `isXxx`, `hasXxx`, `canXxx` — never just `loading` or `open`.

## 16. Accessibility

- Every interactive element is reachable by keyboard. Never `onClick` on a `<div>`.
- Icons accompanying text have `aria-hidden`; icon-only buttons have `aria-label`.
- Form inputs have associated labels. shadcn's `Form` primitives handle this — use them.
- Color is never the only channel of meaning (status badges also have text/icons).
