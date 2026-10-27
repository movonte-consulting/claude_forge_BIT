# Forge cheatsheet

A field guide for Atlassian Forge development. Read this before generating any Forge code.

---

## Runtime

- Always `nodejs22.x` in `manifest.yml`. Node 22 features are available.
- Prefer ES modules. Use `"type": "module"` in `package.json` and `.js` extensions,
  or use `.mjs` files. CommonJS works but the React UI Kit examples are ESM.

---

## `package.json` for modern UI Kit apps — always pin a single React version

**`@forge/react` does NOT re-export `useState`/`useEffect`/`React`.** Its own `index.js`
only exports Forge-specific hooks (`useProductContext`, `useConfig`, etc.) plus
components — see `export * from './components'` in its `index.d.ts`. Standard React
hooks must still come from the `react` package directly:

```jsx
import React, { useState, useEffect } from 'react';
import ForgeReconciler, { Text } from '@forge/react';
```

**Always import `React` itself in every `.jsx` file that contains JSX.** Forge's bundler
uses the classic JSX transform, so `<App />` compiles to `React.createElement(...)`. If
`React` isn't in scope the macro crashes at load with
`Uncaught ReferenceError: React is not defined` — see "Common errors" below. `@forge/react`
does not re-export `React` either, so the default import from `'react'` is required.

**The trap:** `@forge/react` depends on its own `react` + `react-reconciler` +
`react-test-renderer` internally (its custom UI-tree renderer is built on
`react-reconciler`, not `react-dom`). If npm's resolver can't hoist a single shared
`react` for the whole tree, it nests a *second* copy of `react` under
`node_modules/@forge/react/node_modules/react` at a different patch version (e.g.
`18.3.1`) than the one hoisted to the project root (e.g. `18.2.0`). Your frontend code
then imports hooks from one React instance while `@forge/react`'s reconciler renders
with the other. Hook calls into the "wrong" instance find a `null` internal dispatcher.

**Symptom:** the macro deploys and lints cleanly, but throws at runtime —
`Cannot read properties of null (reading 'useState')` — with no code-level cause. This
will NOT show up in `forge lint` or in a code review; it only shows up by inspecting
the installed `node_modules` tree or by actually running the macro.

**Fix — always add this to `package.json` for any modern UI Kit app:**

```json
{
  "overrides": {
    "react": "18.2.0",
    "react-dom": "18.2.0"
  }
}
```

Then do a **clean** reinstall (`rm -rf node_modules package-lock.json && npm install`)
— `npm install` alone or `npm dedupe` on an already-broken tree may leave the stale
nested copy in place. After installing, verify with:

```bash
test -d node_modules/@forge/react/node_modules/react && echo "DUPLICATE — fix needed" || echo "clean"
```

Confirm which exact version to pin by checking `npm ls react --all` first — pin
whichever version the majority of the `@atlaskit/*` tree already resolves to (usually
whatever's already hoisted at the root), not necessarily `18.2.0` — that's the version
that happened to be correct for this project's dependency snapshot at the time this
entry was written, not a hardcoded universal constant.

---

## Two UI patterns (don't mix them up)

Forge has gone through naming churn. As of 2026 there are two patterns in the wild:

### Modern UI Kit (React-based) — RECOMMENDED

Almost always the right choice for new code.

- Components from `@forge/react`
- Frontend renders with `ForgeReconciler.render(<App />)`
- Backend uses resolvers from `@forge/resolver`
- Frontend calls backend via `invoke()` from `@forge/bridge`
- Manifest splits the macro into a `resource` (frontend bundle) and a `resolver.function`
  (backend handler) — these are two separate pieces wired together
- **The macro module needs `render: native` set explicitly.** Without it, current
  Forge CLI (verified on 12.18.0) treats the module as Custom UI by default, and
  lint fails with `Custom UI resource must be a directory` even though the
  `resources[].path` correctly points at a single `.jsx` file. `render: native` is
  what tells the CLI "this is UI Kit" — see `manifest-reference.yml`.

**`@forge/react` component quick-ref (commonly used):**

| Component | Notes |
|---|---|
| `Text` | Inline text |
| `Heading` | `as="h1"` … `"h6"` |
| `Stack` | Vertical layout, `space="space.100"` etc. |
| `Inline` | Horizontal layout |
| `Box` | Generic container |
| `Button` | `onClick` handler |
| `DynamicTable` | **Table component.** Takes `head` + `rows` props — NOT children. `Table`, `Row`, `Head`, `Cell` do NOT exist in `@forge/react`; importing them silently gives `undefined` and renders nothing. |
| `Form`, `TextField`, `Select`, `Checkbox` | Form inputs |
| `Badge`, `Lozenge`, `Tag` | Status indicators |
| `Spinner` | Loading indicator |
| `SectionMessage` | Info/warning/error banners |
| `ModalDialog` | Modal — use `ModalTransition` wrapper |

`DynamicTable` usage:

```jsx
import { DynamicTable } from '@forge/react';

<DynamicTable
  head={{
    cells: [
      { key: 'name', content: 'Name' },
      { key: 'size', content: 'Size' },
    ],
  }}
  rows={items.map((item) => ({
    key: item.id,
    cells: [
      { key: 'name', content: item.name },
      { key: 'size', content: item.size },
    ],
  }))}
/>
```

### Legacy UI Kit (`@forge/ui`)

Older, function-returns-JSX-directly pattern. Avoid unless maintaining an existing app.

- Components from `@forge/ui`
- The function IS the macro — no separate frontend resource
- Manifest macro has `function:` directly (no `resource` / `resolver` split)

**How to tell which a project uses:** check `package.json`. `@forge/react` → modern.
`@forge/ui` → legacy. If both are present, it's a project mid-migration and you
should ask before assuming.

---

## Manifest gotchas

1. **Top-level structure.** `modules`, `resources`, `app`, `permissions` are top-level keys.
   Never nest `resources` or `permissions` under `modules`. This is the most common
   hallucination.

2. **Function handler paths.** `handler: <file>.<exportedFunctionName>` — relative to
   project root (or `src/` depending on bundler config). For a function exported as
   `handler` from `src/index.js`, write `handler: index.handler`.

3. **App ID format.** `ari:cloud:ecosystem::app/<UUID>`. Generated by `forge register`.
   Don't hand-edit.

4. **Macro `resource` vs `resolver.function`.** In modern UI Kit, these are TWO separate
   things:
   - `macro.resource` → points to a key in the top-level `resources` array → that key
     maps to a path like `src/frontend/index.jsx` (the frontend bundle)
   - `macro.resolver.function` → points to a key in the top-level `function` array →
     that key has a `handler: index.handler` (the backend resolver)

5. **Permissions are runtime-enforced.** `permissions.scopes` must explicitly include
   what you call. Missing scopes mean a successful `forge deploy` but 401s at runtime.
   `read:page:confluence` for v2 page reads, `read:attachment:confluence` for attachments.
   **Listing a page's attachments (`/wiki/api/v2/pages/{id}/attachments`) needs ALL THREE
   scopes: `read:page:confluence` + `read:attachment:confluence` + `read:confluence-user`
   (the last only if you resolve uploader names).** Omitting `read:page:confluence` because
   "I only read attachments" gives an API error at runtime (deploy and lint both pass).
   Always start from the scope list in `manifest-reference.yml` and only remove scopes
   deliberately.
   For `GET /wiki/rest/api/user` (v1 user lookup by accountId), the required scope is
   **`read:confluence-user`** — NOT `read:user:confluence`. `forge lint` will catch a
   wrong/missing scope with a `permission-scope-required` error naming the exact scope
   it expects; trust that error over guessing, and cross-check against
   `scopes-product-confluence` in `llms.txt` if unsure.

6. **Whitespace and indentation.** YAML is whitespace-sensitive. Use 2 spaces. Mixing
   tabs and spaces will fail validation with a cryptic message.

---

## `@forge/api` essentials

```js
import api, { route } from '@forge/api';

// asUser: respects the calling user's permissions (default for most macros)
const res = await api.asUser().requestConfluence(
  route`/wiki/api/v2/pages/${pageId}/attachments`,
  { headers: { Accept: 'application/json' } }
);

// asApp: runs with the app's permissions
// Use for cross-user data, scheduled tasks, or app-identity operations
const res = await api.asApp().requestConfluence(
  route`/wiki/rest/api/user?accountId=${accountId}`,
  { headers: { Accept: 'application/json' } }
);
```

**Rules:**

- **ALWAYS use the `` route`...` `` tagged template.** It handles URL encoding for path
  segments. Plain template strings will break on any ID containing special characters.
- **Default to `asUser()`.** Switch to `asApp()` only when you genuinely need the app's
  identity. Misusing `asApp()` is a common security-review finding.
- **`requestConfluence` for Confluence, `requestJira` for Jira.** Don't reach for raw
  `fetch` for Atlassian APIs — the wrapper handles auth.
- **Always check `res.ok`** before calling `res.json()`. Failed Atlassian responses are
  often HTML error pages, which throw a confusing JSON parse error.

---

## Resolvers (modern UI Kit backend)

**Use `makeResolver` — not `new Resolver()`.** `@forge/resolver` exports `Resolver` as
`exports.default`, but the Forge bundler does not apply ESM/CJS default-import interop,
so `import Resolver from '@forge/resolver'` receives the whole module object and
`new Resolver()` throws "is not a constructor". `makeResolver` is a named export and
always works.

```js
import { makeResolver } from '@forge/resolver';
import api, { route } from '@forge/api';

export const handler = makeResolver({
  getThing: async ({ context, payload }) => {
    // context.extension contains module-specific context
    // For a Confluence macro: context.extension.content.id is the page ID
    // For a Jira issue panel: context.extension.issue.id is the issue ID
    const pageId = context.extension?.content?.id;
    // ...do work, return JSON-serializable result
  },
});
```

The frontend calls these by name. **Always add `.catch()`** — if the promise rejects
(permission error, resolver crash, etc.) and there is no catch handler, the UI will
be stuck on its loading state indefinitely:

```jsx
import { invoke } from '@forge/bridge';

invoke('getThing', { someArg: 'value' })
  .then((result) => { /* update state */ })
  .catch((err) => { /* set error state — never swallow this */ });
```

---

## Common errors and what they really mean

| Error                                                | Reality                                                                                                                                  |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `Cannot find module './index'` at deploy             | Handler path in `manifest.yml` doesn't match the actual file or export name. Check `handler: index.handler` matches what `src/index.js` actually exports. |
| `Unauthorized` / 401 at runtime                      | Missing scope in `permissions.scopes`. Deploy passes; runtime fails when the call is made.                                                                |
| `Validation error: function key '...' not found`     | A `resolver.function` reference points to a key not declared in the top-level `function` array.                                                            |
| `Tunnel could not be established` on `forge tunnel`  | Local Node version mismatch with manifest runtime. Use `nvm use 22` to match `nodejs22.x`.                                                                 |
| Macro renders empty / blank                          | (1) Frontend `resource.path` is wrong, (2) `ForgeReconciler.render()` isn't called, (3) resolver returned `undefined`, or **(4) you used `Table`/`Row`/`Head`/`Cell` which don't exist in `@forge/react` — use `DynamicTable` instead**. |
| `invoke()` stuck on loading state forever            | No `.catch()` on the `invoke()` promise. A rejected promise silently leaves state unchanged. Always add `.catch((err) => setState({ error: ... }))`. |
| `Invocation failed` with no details                  | Run `forge logs -s 5m`. Resolver threw synchronously before returning. Wrap in try/catch and return `{ error: err.message }`.                             |
| `Resolver is not a constructor`                      | Using `import Resolver from '@forge/resolver'` — Forge's bundler doesn't apply CJS default-import interop. Switch to `import { makeResolver } from '@forge/resolver'`. |
| `Custom UI resource must be a directory` (lint)      | Macro module is missing `render: native`. Without it, the CLI defaults to treating the resource as Custom UI, which requires a directory bundle instead of a single `.jsx` entry file. Add `render: native` next to `resource:` in the macro module. |
| `permission-scope-required` naming a scope you don't recognize | Trust the lint error's exact scope string over the cheatsheet — scope names occasionally differ from the obvious guess (e.g. the v1 user-lookup endpoint needs `read:confluence-user`, not `read:user:confluence`). Cross-check `scopes-product-confluence` in `llms.txt` if unsure. |
| Lint error: file not found (when file clearly exists)| Path in `manifest.yml` or `package.json main` doesn't match the actual file location. Forge CLI lints both.                                                |
| `Uncaught ReferenceError: React is not defined` in the browser console (spinner forever, `forge logs` empty) | A `.jsx` file with JSX is missing the default `React` import. The frontend crashes at load, before `invoke()` is ever called, so the resolver never runs and there is nothing in `forge logs`. Use `import React, { useState, useEffect } from 'react';`. Deploy + lint both pass. |
| `Cannot read properties of null (reading 'useState')` at runtime (deploy + lint both pass) | Two copies of `react` in `node_modules` — one hoisted at root, one nested under `@forge/react/node_modules/react` at a different version, pulled in by `@forge/react`'s internal `react-reconciler`/`react-test-renderer`. Pin `react`/`react-dom` via `overrides` in `package.json` and do a clean `rm -rf node_modules package-lock.json && npm install`. See "package.json for modern UI Kit apps" section above. |

---

## Confluence Cloud API quick refs

- **List page attachments:** `GET /wiki/api/v2/pages/{id}/attachments`
- **Get page content:** `GET /wiki/api/v2/pages/{id}?body-format=storage`
- **Get user by accountId:** `GET /wiki/rest/api/user?accountId={id}` (still v1)
- **Search content:** `GET /wiki/api/v2/pages?title={title}&space-id={spaceId}`

**Pagination:** never interpolate the whole `_links.next` into `route` (it percent-encodes
the path). Extract the `cursor` query param from it and pass it as `` route`...&cursor=${cursor}` ``,
and omit `cursor` entirely on the first request — an empty `cursor=` can return a 400.

v2 responses are shaped `{ results: [...], _links: { next?: "..." } }`. Paginate by
following `_links.next` if present.

---

## Deploy + install workflow

```bash
forge register             # one-time per app, gets app ID
forge deploy               # builds + uploads to development environment
forge install              # installs to a site (first time)
forge install --upgrade    # subsequent deploys
forge logs -s 5m -n 50     # recent logs (--tail does not exist)
forge tunnel               # local dev with hot reload
```

**Gotcha:** after `forge deploy`, the update is NOT live on the installed site until
you run `forge install --upgrade`. Forgetting this is the #1 "why isn't my change
showing up" cause.

---

## Debugging a macro stuck on a spinner

`forge logs` only shows **backend** (resolver) invocations. Triage by where the failure is:

- **`forge logs -s 5m` is empty after reloading the page** → the frontend crashed (or an
  old version is installed) before `invoke()` ran. Open the browser DevTools console and
  select the macro's iframe in the **JavaScript context** dropdown (it is not `top`) —
  errors there never reach `forge logs`. Typical culprits: missing `React` import,
  duplicate `react` copies, importing a component that doesn't exist in `@forge/react`.
  Also confirm you ran `forge install --upgrade` after `forge deploy`.
- **`forge logs` shows an error** → the resolver ran and threw. Check scopes in
  `permissions.scopes` and wrap resolver bodies in try/catch.
- Ignore unrelated console noise from Confluence itself (Sentry 429s, `gateway/...` 404/403s,
  `child/attachment` 401s, `unload` permissions-policy warnings).
- Export the console via right-click → **Save as…** to share the log.

---

## When in doubt

- Open `manifest-reference.yml` in this folder — it's a known-good annotated example.
- Fetch the relevant URL from `llms.txt` rather than guessing API shapes.
- If you're proposing a structural change to `manifest.yml`, show me the full
  resulting file, not just the diff. Manifest errors are positional and easy to miss
  in a partial view.
