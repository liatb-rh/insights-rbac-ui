# Agent instructions: Cost on‑prem host × insights‑rbac‑ui (Module Federation)

Use this document when integrating **insights‑rbac‑ui** into the **Cost on‑prem** shell as a **micro frontend (MFE)** via webpack **Module Federation**, including exposing **narrow internals** (e.g. roles). Prefer **minimal changes** in this repo and **no fork** unless the host cannot satisfy the contracts below.

**Related:** Product/architecture detail lives in `[on-prem-plan.md](./on-prem-plan.md)`.

---

## 1. Goals (what “done” means)

1. **Cost on‑prem** loads RBAC UI chunks from the RBAC deployment (`fed-mods.json` + remote entry) like SaaS Scalprum does.
2. The host can embed either `**./Iam`** (full IAM) or **narrow remotes** such as `**./modules/RolesAccessManagement`** (and future `./modules/…` slices).
3. **insights‑rbac‑ui** keeps **one route definition** per slice: extract `**v2*RouteElements`** from `V2Routing`, never duplicate `<Route>` trees between host and `Routing.tsx`.
4. **Cost on‑prem** uses a **separate build artifact** when product requires stripped UX (see `on-prem-plan.md` §5); agents wire **build args**, not runtime forks in the host.

---

## 2. Non‑negotiables (repository rules)


| Rule                                                                                                                                                             | Why                                                                                            |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| New federation entry files live under `**src/federated-modules/`** only                                                                                          | ESLint boundary: only `Iam.tsx` and `federated-modules/` may import from `src/v1/` / `src/v2/` |
| Narrow slices reuse `**IamSharedProviders**` (`src/shared/providers/IamSharedProviders.tsx`) when the UI calls `**useAppServices()**`, React Query, or RBAC APIs | Avoids broken data layer in remotes                                                            |
| V2 Kessel surfaces wrap `**AccessCheck.Provider**` the same way as `**IamV2**`                                                                                   | `useRolesAccess` / `v2Guard` require Kessel                                                    |
| Routes for a remote **must** come from `**src/v2/routes/v2<Slice>RouteElements.tsx`** consumed by `**V2Routing**`                                                | Single source of truth; prevents drift                                                         |
| Register every expose in `**fec.config.js**` under `**moduleFederation.exposes**`                                                                                | Or the remote is absent from `**fed-mods.json**`                                               |


---

## 3. Host (Cost on‑prem) responsibilities

### 3.1 Module Federation / Scalprum

1. Register the `**rbac**` application with the **manifest URL** your deployment serves (SaaS pattern: `/apps/rbac/fed-mods.json` — align with your ingress).
2. Ensure `**react-router-dom`** is shared per RBAC’s federation config (**singleton**, compatible **v6** range) so `Link` / `Outlet` match the host router.
3. Load UI with `**AsyncComponent`** from `**@redhat-cloud-services/frontend-components**` (or your platform’s equivalent that resolves Scalprum scopes):

```tsx
import { AsyncComponent } from '@redhat-cloud-services/frontend-components';

// Example: full IAM
<AsyncComponent appName="rbac" module="./Iam" fallback={<Spinner />} />

// Example: roles only (when expose exists in built fed-mods.json)
<AsyncComponent appName="rbac" module="./modules/RolesAccessManagement" fallback={<Spinner />} />
```

Use `scope` instead of `appName` only if your dependency major/version requires it.

### **3.2 React Router**

- Mount descendants under `basename="/iam"` unless the program explicitly changes `useAppLink` / `pathnames` (high‑cost change — avoid).
- **Relative paths** inside RBAC assume `/iam/access-management/…`, `/iam/overview`, etc.

### **3.3 Platform APIs**

- **RBAC API** and **Kessel** (`/api/kessel/v1beta2` relative to origin from RBAC’s `AccessCheck.Provider`) must be reachable from the browser through your gateway.
- Provide **Insights‑compatible** `useChrome` / auth identity hooks **or** approved shims so `IamSharedProviders` receives `getToken`, `environment`, `identity`.

### **3.4 Feature flags**

- SaaS: Unleash context.
- Cost on‑prem: prefer **build‑time** behavior split in **insights‑rbac‑ui** (`RBAC_UI_DEPLOYMENT=cost-onprem`) plus host‑injected static flags if components still call `useFlag`.

---

## **4. insights‑rbac‑ui: minimal‑change workflow for a new narrow remote**

**Reference implementation:** `RolesAccessManagement` + `v2RolesRouteElements`.

### **Step A — Extract routes**

1. Identify the contiguous `<Route>…</Route>` block in `src/v2/Routing.tsx` for the slice (roles, org management, users, etc.).
2. Move it to `src/v2/routes/v2<Slice>RouteElements.tsx` exporting `v2<Slice>RouteElements` as a **React fragment** (`<> … </>`) of `Route` nodes (same lazy imports, `v2Guard`, `outletElement` as today).
3. Replace the original block in `V2Routing.tsx` with `{v2<Slice>RouteElements}`.

### **Step B — Federated shell**

Create `src/federated-modules/<Slice>Federated.tsx` (copy `RolesAccessManagement.tsx` structure):

1. `IntlProvider` + locale messages (same as `Iam`).
2. `NotificationsProvider` + store dismiss delay if mutations show toasts.
3. `IamSharedProviders` (`testMode` prop optional for Storybook).
4. `AccessCheck.Provider` with `baseUrl={window.location.origin}` and `apiPath='/api/kessel/v1beta2'` when the slice uses Kessel.
5. `Suspense` + `AppPlaceholder` fallback.
6. Inner `Routes>{v2<Slice>RouteElements}</Routes>`.

### **Step C — Webpack expose**

In `fec.config.js`:

'./modules/<Slice>Federated': path.resolve(__dirname, './src/federated-modules/<Slice>Federated.tsx'),

### **Step D — Verify on‑prem image**

- Confirm **cost‑on‑prem** CI passes `fed-mods.json` containing the new `./modules/…` key before the host references it.

### **Step E — Storybook + docs**

- Add `src/federated-modules/<Slice>Federated.stories.tsx`: `parameters.noWrapping: true`, `MemoryRouter basename="/iam"`, `initialEntries` under the slice URL, MSW handlers from `src/v2/data/mocks/` (no inline MSW — see `AGENTS.md`).
- Extend `src/docs/ModuleFederation.mdx` (or `ExposingFeatureModulesFederation.mdx`) with the expose name and host prerequisites.

---

## **5. Anti‑patterns (do not do)**


| **Anti‑pattern**                                                            | **Problem**                                                  |
| --------------------------------------------------------------------------- | ------------------------------------------------------------ |
| Point `exposes` directly at `src/v2/features/…`                             | Breaks boundary rules; skips provider stack                  |
| Copy‑paste routes only inside `federated-modules`                           | Drifts from `V2Routing`                                      |
| Nested `BrowserRouter` inside remote while host already has Router          | Double router / broken `useNavigate`                         |
| Omit `IamSharedProviders` for slices that use `useRolesV2Query` / mutations | Runtime failures                                             |
| Assume host Redux feeds RBAC                                                | RBAC uses `ServiceProvider` + TanStack Query, not host Redux |


---

## **6. Quick reference — canonical paths (basename** `/iam`**)**


| **Slice**                   | **Example URL**                                                        |
| --------------------------- | ---------------------------------------------------------------------- |
| Overview                    | `/iam/overview`                                                        |
| My Access                   | `/iam/my-access`, `/iam/my-access/groups`, `/iam/my-access/workspaces` |
| Users                       | `/iam/access-management/users-and-user-groups/users`                   |
| Roles                       | `/iam/access-management/roles`, …/add-role, …/edit/:roleId             |
| User groups (“Cost groups”) | `/iam/access-management/users-and-user-groups/user-groups`             |


---

## **7. Agent checklist before opening a PR**

- `v2<Slice>RouteElements` extracted; `V2Routing` imports fragment — diff proves no behavioral change for SaaS.
- Federated file under `src/federated-modules/` only.
- `fec.config.js` updated; local build lists new key in `fed-mods.json`.
- Storybook federated story + `noWrapping` where providers are self‑contained.
- `AGENTS.md` non‑negotiables (MSW factories, Storybook imports) respected.
- Cost on‑prem: if UX is stripped, changes gated by **build‑time** constant per `on-prem-plan.md`, not scattered magic strings.

---

## **8. Escalation**

- **Kessel unavailable on‑prem:** Permission model may require platform/backend decision — do not silently remove guards without security review.
- **Chrome incompatible:** Shim layer belongs in **host** or a thin **adapter package** — avoid rewriting `IamSharedProviders` per consumer unless approved.

