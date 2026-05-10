# **Plan: Module federation–first RBAC UI for SaaS and Cost on‑prem**

## **1. Motivation**

### **1.1 Business / product drivers**

- **Multiple hosts:** Red Hat ships IAM/RBAC UX in **Hybrid Cloud Console (SaaS)** and in **Cost on‑prem** (and related shells). Both need RBAC capabilities without maintaining divergent forks long term.
- **Composable UX:** Product wants to embed **narrow slices** of IAM—e.g. **roles**, **organization management**—inside parent experiences (navigation islands, settings areas, or partner consoles) rather than always loading the full `./Iam` application.
- **Clear contracts:** Exposing **stable federated entry points** (webpack Module Federation remotes documented in `fed-mods.json`) gives platform teams a **versioned integration surface** (`AsyncComponent` + `appName` + `module` path).

### **1.2 Technical drivers**

- **Independent deploy:** `insights-rbac-ui` already ships as its own Frontend asset bundle (`/apps/rbac`, `fed-mods.json`). Federation aligns with that model.
- **Shared runtime expectations:** Narrow remotes still need **auth, axios, React Query, Kessel (where used), Router basename** `/iam`**, Unleash/static config**—centralizing provider stacks avoids each consumer re‑implementing chrome wiring.
- **Cost on‑prem constraints:** On‑prem often prefers **build‑time product splits** (separate artifact per deployment flavor), simpler operations than runtime feature matrices, and **smaller behavioral surface** (narrow nav + removals). Federation lets **the same codebase** produce **SaaS vs cost‑on‑prem bundles** with different exposed modules and/or internal route sets.

---

## **2. Current technical baseline (short)**

- **Build:** `@redhat-cloud-services/frontend-components-config` + `fec.config.js` → `moduleFederation.exposes` → `fed-mods.json` consumed by the shell (Scalprum pattern).
- **Full app:** `./Iam` federated entry wraps notifications, `IamSharedProviders` (services + React Query + API errors), then V1/V2 routing.
- **Pattern established:** `RolesAccessManagement` remote bundles `IamSharedProviders` **+ Kessel** `AccessCheck.Provider` **+** `Routes` **+ shared** `v2RolesRouteElements` so roles list/add/edit stay in sync with `V2Routing`.
- **Shared route fragments:** `src/v2/routes/v2RolesRouteElements.tsx` avoids duplicating route definitions between monolith and remote.

---

## **3. Target architecture**

### **3.1 Principles**

1. **One codebase, multiple artifacts:** SaaS build vs Cost on‑prem build differ by **environment / webpack defines** (see §5), not by maintaining two repos.
2. **Narrow remotes = provider shell + route/component fragment:** Each remote is a `src/federated-modules/<Name>.tsx` entry that:
  - Reuses `IamSharedProviders` (and **Kessel** when V2 guards/hooks require it),
  - Optionally adds **notifications / i18n** like `Iam`,
  - Renders `Routes` **+ shared fragment** or a **single page component** when routing is trivial.
3. **Single source of truth for routes:** Any federated slice that mirrors in‑app navigation must **import the same route JSX fragment** used by `V2Routing` (same pattern as roles).
4. **Documentation + Storybook:** Each new remote gets **Storybook** coverage (`noWrapping` when self‑wrapped) and an entry in `ModuleFederation.mdx` / `ExposingFeatureModulesFederation.mdx` (or successor).

### **3.2 What “narrow objects” means in code**


| **Slice**               | **Suggested remote name (example)**                               | **Notes**                                                                                                                                |
| ----------------------- | ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Roles                   | `./modules/RolesAccessManagement`                                 | Extract                                                                                                                                  |
| Organization management | `./modules/OrganizationManagementAccess` (TBD)                    | Extract `v2OrgManagementRouteElements` from `V2Routing`; wrap with same provider stack as roles; match guards (`v2GuardOrgAdmin`, etc.). |
| Future (optional)       | `./modules/UsersAndGroupsAccess`, `./modules/MyAccessShell`, etc. | Only if product requires embedding without `./Iam`.                                                                                      |


## **4. Implementation guide (how to add a narrow remote)**

### **4.1 Steps (repeatable)**

1. **Extract routes** from `src/v2/Routing.tsx` into `src/v2/routes/v2<Slice>RouteElements.tsx` (fragment of `<Route>` children, same lazy components and guards).
2. **Replace** the inlined block in `V2Routing.tsx` with `{v2<Slice>RouteElements}`.
3. **Add** `src/federated-modules/<Slice>Federated.tsx`:
  - `IntlProvider` + `NotificationsProvider` (match `Iam` if mutations/toasts needed),
  - `IamSharedProviders`,
  - `AccessCheck.Provider` if slice uses Kessel (`useRolesAccess`, org admin guards, etc.),
  - `Suspense` + `Routes` wrapping the fragment.
4. **Register** in `fec.config.js`: `'./modules/<Slice>Federated': path.resolve(...)`.
5. **Document** consumer snippet (`AsyncComponent`, `basename="/iam"` requirement).
6. **Storybook:** `Federated Modules/<Slice>` with `MemoryRouter basename="/iam"` + MSW + `noWrapping: true` where appropriate.
7. **Frontend CRD / host manifest:** Optional second `modules[]` entry in `deploy/frontend.yaml` **only if** the platform registers a dedicated pathname; many hosts embed via `AsyncComponent` without a new CRD row.

### **4.2 Consumer contract (both platforms)**

- Host provides **React Router v6** with `basename="/iam"` (unless the team agrees to change `pathnames` / `useAppLink`—heavier change).
- Host provides **Insights‑compatible chrome hooks** (or shims): auth token, identity, environment—or fork replaces those hooks for on‑prem (outside this doc).
- **Kessel:** present when remote uses V2 domain hooks/guards; otherwise remote will fail permission checks or API calls.

---

## **5. Cost on‑prem vs SaaS (separate build env)**

### **5.1 Motivation for build‑time split**

- Avoid reliance on Unleash topology on disconnected installs.
- **Image = behavior** for operations.
- Allows **excluding** SaaS‑only routes/UI from the on‑prem bundle (optional tree‑shaking / clearer attack surface).

### **5.2 Implementation sketch**

1. **Single env variable** at build time, e.g. `RBAC_UI_DEPLOYMENT=saas | cost-onprem` (exact naming for team agreement).
2. **Webpack DefinePlugin** (via `fec.config.js` `plugins`) to inject a compile‑time constant consumed by `src/shared/deployment.ts` (`isCostOnpremDeployment()`).
3. **Conditional composition:**
  - **On‑prem:** `V2Routing` renders `v2CostOnPremRouteElements` only (overview, my access, users, groups, roles, identity providers, org management if in scope—per product spec).
  - **SaaS:** unchanged full `V2Routing`.
4. **Conditional** `fec.config.js` **exposes:** e.g. on‑prem build additionally or exclusively exposes narrowed remotes required by Cost shell; SaaS keeps full set.
5. **CI:** matrix build → two container tags (`…-saas`, `…-cost-onprem`).

### **5.3 UX simplifications (product spec, behind cost‑on‑prem build only)**

- Overview: strip **recommended content**, **HCC docs / learning links** (compile‑time branch in `shared/components/overview/Overview.tsx` or wrapper).
- Users: remove **invite users** actions and routes from **cost‑on‑prem route subset**.
- Groups: remove **add service accounts** entry points from **cost‑on‑prem** build + exclude routes.



---

## **6. Work breakdown (insights‑rbac‑ui repo)**


| **Area**                           | **Work items**                                                                                                                                                                         |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Federation infrastructure**      | Document pattern; keep `IamSharedProviders` as shared shell; optional second doc page for “org management remote.”                                                                     |
| **Organization management remote** | Extract `v2OrganizationManagementRouteElements`; add `OrganizationManagementFederated.tsx`; `fec.config.js`; Storybook; MSW as needed.                                                 |
| **Cost on‑prem**                   | `deployment.ts` + webpack define; `v2CostOnPremRouteElements`; conditional `V2Routing`; Overview/Users/Groups tweaks; new **Identity Providers** page + API client when backend ready. |
| **Testing**                        | Story variants per deployment; Playwright smoke if pipeline covers federated mounts.                                                                                                   |
| **Release**                        | Two artifacts; runbook for which image goes to which cluster; `fed-mods.json` diff checklist.                                                                                          |


---

## **7. Risks and mitigations**


| **Risk**                           | **Mitigation**                                                                              |
| ---------------------------------- | ------------------------------------------------------------------------------------------- |
| Route drift between app and remote | Shared `v2*RouteElements` only—no copy‑paste.                                               |
| Missing providers in remote        | Mirror `RolesAccessManagement` stack; lint or checklist in PR template.                     |
| Kessel absent on‑prem              | Spike early; may need alternate guards or backend‑proxy behavior—product/security decision. |
| Two builds = double QA             | Automated smoke on both artifacts; shared integration tests where possible.                 |


---

## **8. Success criteria**

- SaaS hosts can load `./Iam` unchanged and optionally `./modules/<Slice>` for embedded experiences.
- Cost on‑prem hosts deploy **cost‑on‑prem image**, load agreed remotes, **without** SaaS‑only UX paths exposed.
- Each new remote is **documented** with host prerequisites (`basename`, chrome, Kessel) and has **Storybook** coverage.



# **Technical Plan: Exposing Overview, My Access, Users, Roles, and Groups via Module Federation (SaaS + Cost on‑prem)**



## **1. Motivation (for engineers)**

- **SaaS** and **Cost on‑prem** hosts need to embed **narrow IAM slices** without loading full `./Iam` or duplicating RBAC logic.
- **Module Federation** exposes webpack remotes listed in `fed-mods.json`; hosts load them with `AsyncComponent` (`@redhat-cloud-services/frontend-components`) using `appName="rbac"` (or the configured scope) and `module="./modules/<ExportName>"`.
- **Minimal drift:** Route trees used inside `./Iam` and inside remotes should come from **shared** `v2*RouteElements` **fragments** (same approach as roles — see `src/v2/routes/v2RolesRouteElements.tsx` + `src/federated-modules/RolesAccessManagement.tsx`).

---

## **2. Global specifications (apply to every remote)**

### **2.1 URL model**

- Browser URL prefix: `/iam` = React Router `basename`.
- `pathnames` in `src/v2/utilities/pathnames.ts` define segments **relative to** `/iam` (e.g. `/overview/`*, `/access-management/roles/*`).

**Examples (full paths hosts link to):**


| **Area**             | **Typical entry URL**                                                         |
| -------------------- | ----------------------------------------------------------------------------- |
| Overview             | `/iam/overview`                                                               |
| My User Access       | `/iam/my-access` (tabs: `/iam/my-access/groups`, `/iam/my-access/workspaces`) |
| Users                | `/iam/access-management/users-and-user-groups/users`                          |
| Roles                | `/iam/access-management/roles`                                                |
| Groups (user groups) | `/iam/access-management/users-and-user-groups/user-groups`                    |


### **2.2 Host prerequisites**


| **Requirement**       | **Detail**                                                                                                               |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Router                | React Router **v6**, `basename="/iam"`                                                                                   |
| Chrome / platform     | Hooks used by `IamSharedProviders` (`usePlatformAuth`, `usePlatformEnvironment`, `useIdentity`) must resolve             |
| Unleash or equivalent | Features using `useFlag` need a provider (SaaS Unleash; on‑prem may inject static flags per separate‑build plan)         |
| Kessel                | Any slice using `v2Guard` / `useRolesAccess` / tenant hooks needs `AccessCheck.Provider` and `/api/kessel/...` reachable |
| Notifications         | If slice uses toasts/mutations via `useAddNotification`, wrap with `NotificationsProvider` (see `RolesAccessManagement`) |


### **2.3 Consumer pattern (example)**

import { AsyncComponent } from '@redhat-cloud-services/frontend-components';

*// Host tree must include BrowserRouter basename="/iam" (or equivalent)*

<AsyncComponent

appName="rbac"

module="./modules/RolesAccessManagement"

fallback={<Spinner />}

/>

*(Adjust* `scope` *vs* `appName` *per your* `@redhat-cloud-services/frontend-components` *major version.)*

### **2.4 Repo checklist for any new remote**

1. `src/v2/routes/v2<Slice>RouteElements.tsx` — JSX fragment of `<Route>...</Route>` copied from `V2Routing`, **zero behavior change**.
2. `src/v2/Routing.tsx` — Replace inlined block with `{v2<Slice>RouteElements}`.
3. `src/federated-modules/<Slice>Federated.tsx` — Provider shell + `<Routes>{v2<Slice>RouteElements}</Routes>` (+ `Suspense`).
4. `fec.config.js` — `'./modules/<Slice>Federated': path.resolve(__dirname, './src/federated-modules/<Slice>Federated.tsx')`.
5. **Storybook** — `Federated Modules/<Slice>`; `parameters.noWrapping: true` when the remote bundles its own providers; `MemoryRouter basename="/iam"` + `initialEntries` under test path; **MSW** aligned with feature handlers.
6. **Docs** — `ModuleFederation.mdx` table row + link to integration notes.

---

## **3. Per‑surface specifications**

Below: **what it is in code**, **routes to extract**, **provider stack**, **federated module shape**, **on‑prem (cost) notes**.

---

### **3.1 Overview**

**Purpose:** Landing content for IAM / Access Management overview.

**Implementation today**

- Route: `V2Routing` mounts `pathnames.overview.path` → `/overview/*` with `v2Guard([roles.canView])`.
- Lazy component: `./features/overview/Overview` → thin wrapper around `src/shared/components/overview/Overview.tsx` (links, flags, recommended content, external links).

**Routes to share (*`*v2OverviewRouteElements` **— proposed name)**

*// Pseudocode — mirror V2Routing exactly*

<Route {...v2Guard([roles.canView])}>

<Route path={pathnames.overview.path} element={<V2Overview />} />

</Route>

**Provider stack (recommended)**


| **Provider**            | **Needed?** | **Why**                                                                                                  |
| ----------------------- | ----------- | -------------------------------------------------------------------------------------------------------- |
| `IntlProvider`          | Yes         | `useIntl`                                                                                                |
| `NotificationsProvider` | Optional    | Only if overview actions ever toast                                                                      |
| `IamSharedProviders`    | Optional*   | Overview subtree historically light on React Query; include if any child starts calling `useAppServices` |
| `AccessCheck.Provider`  | Yes         | `v2Guard` / Kessel                                                                                       |
| Router                  | Host        | `AppLink` uses `Link`                                                                                    |


* *Minimal remote:* match `V2Overview` **federated spike**: `IntlProvider` **+** `AccessCheck` **+ page**. Add `IamSharedProviders` if you unify all remotes on one stack for consistency.

**Federated entry (example)**

- File: `src/federated-modules/OverviewAccessManagement.tsx` (name TBD).
- Export: default component wrapping `IntlProvider` **→** `AccessCheck.Provider` **→** `Suspense` **→** `Routes` **→** `{v2OverviewRouteElements}`.
- `fec.config.js`**:** `'./modules/OverviewAccessManagement': ...`.

**Cost on‑prem**

- **Separate build:** use `RBAC_UI_DEPLOYMENT=cost-onprem` (or equivalent) to render `Overview` variant without `RecommendedContentTable`, HCC doc `linkProps`, and `console.redhat.com` learning link — either props on `Overview` or a wrapper `CostOnPremOverview` imported only in cost build.

**Engineering example — host deep link**

/iam/overview

---

### **3.2 My User Access**

**Purpose:** End‑user view of **their** access (groups, workspaces tabs).

**Implementation today**

- Parent: `pathnames['my-access'].path` → `/my-access/`*, element `<MyAccess />`.
- Children: `MyGroups`, `MyWorkspaces` under relative paths `groups/*`, `workspaces/*` (`pathnames` lines 31–40).

**Routes fragment (*`*v2MyAccessRouteElements`**)**

<Route path={pathnames['my-access'].path} element={<MyAccess />}>

<Route path={pathnames['my-access-groups'].path} element={<MyGroups />} />

<Route path={pathnames['my-access-workspaces'].path} element={<MyWorkspaces />} />

</Route>

**Provider stack**


| **Provider**            | **Needed?**                  | **Why**                                                                                                                               |
| ----------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `IntlProvider`          | Yes                          | i18n                                                                                                                                  |
| `NotificationsProvider` | Likely yes                   | Patterns elsewhere use toasts for errors                                                                                              |
| `IamSharedProviders`    | **Yes**                      | My Access pages typically hit **identity-backed APIs** via shared data hooks                                                          |
| `AccessCheck.Provider`  | **Confirm with code review** | If routes are “public” but children use Kessel, mirror `IamV2`; if only chrome identity, may still need `AccessCheck` for consistency |


**Federated entry**

- `src/federated-modules/MyAccessManagement.tsx` → same shell pattern as `RolesAccessManagement` if `AccessCheck` required:  
`IntlProvider` **→** `NotificationsProvider` **→** `IamSharedProviders` **→** `AccessCheck.Provider` **→** `Suspense` **→** `Routes` **→** `{v2MyAccessRouteElements}`.

**Cost on‑prem**

- Include fragment in **cost‑on‑prem route bundle** if product ships “My User Access” on‑prem.
- If **workspaces** tab is out of scope for Cost, **omit** `my-access-workspaces` route from `v2MyAccessRouteElements` only in cost build (compile‑time constant).

**Example URLs**

/iam/my-access

/iam/my-access/groups

/iam/my-access/workspaces

---

### **3.3 Users (Access Management — principals list)**

**Purpose:** Org admins manage **users** (principals) under Access Management.

**Implementation today**

- Nested under `pathnames['users-and-user-groups'].path` with `UsersAndUserGroups` layout.
- Users list: `pathnames['users-new'].path` → `AccessManagementUsers` (`.../users/Users.tsx`).
- Invite modal route: `invite` child under users (`InviteUsersModalCommonAuth`) behind `v2GuardOrgAdmin()`.

**Minimal federation slice**

- Option **A (narrow):** Extract **only** users subtree:

*// Inside broader guard matching V2Routing*

<Route {...v2Guard([principals.canList, groups.canView], { checkAll: false })}>

<Route path={pathnames['users-and-user-groups'].path} element={<UsersAndUserGroups />}>

<Route {...v2Guard([principals.canList])}>

<Route path={pathnames['users-new'].path} element={<AccessManagementUsers />}>

{*/* cost-onprem build: omit invite child entirely */*}

<Route {...v2GuardOrgAdmin()}>

<Route path={pathnames['invite-group-users'].path} element={outletElement(InviteUsersModalCommonAuth)} />

</Route>

</Route>

</Route>

{*/* groups sibling omitted if truly users-only remote */*}

</Route>

</Route>

- Option **B:** Ship **“Users & User Groups”** shell remote so **tabs** match production (`UsersAndUserGroups`).

**Provider stack**


| **Provider**                                                                          | **Needed?**                                                      |
| ------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| `IntlProvider`, `NotificationsProvider`, `IamSharedProviders`, `AccessCheck.Provider` | **Yes** (same class as roles — list + mutations + Kessel guards) |


**Cost on‑prem**

- **Compile‑time:** drop `invite` `<Route>` and hide invite actions in `UsersTable` / `Users.tsx` for cost build only.

**Example URL**

/iam/access-management/users-and-user-groups/users

---

### **3.4 Roles**

**Purpose:** V2 roles CRUD (list, add wizard, edit).

**Status**

- **Canonical pattern already implemented** in-repo:
  - `src/v2/routes/v2RolesRouteElements.tsx`
  - `src/federated-modules/RolesAccessManagement.tsx` — `IntlProvider` → `NotificationsProvider` → `IamSharedProviders` → `AccessCheck.Provider` → `Suspense` → `Routes` **→** `{v2RolesRouteElements}`

`fec.config.js` **example**

'./modules/RolesAccessManagement': path.resolve(__dirname, './src/federated-modules/RolesAccessManagement.tsx'),

**Example URLs**

/iam/access-management/roles

/iam/access-management/roles/add-role

/iam/access-management/roles/edit/:roleId

**Cost on‑prem**

- Same remote in cost image **if** roles ship; ensure **Kessel + RBAC API** exist on‑prem.

---

### **3.5 Groups (Cost: IDP‑backed groups for role assignment)**

**Product mapping**

- In **this codebase**, the closest match is **Access Management “User groups”**: `AccessManagementUserGroups` (`user-groups/`*), plus **create / edit** routes (`AddGroupWizard`, `EditUserGroup`, `CreateUserGroup`).
- **Naming:** Cost may call these “groups”; RBAC UI still uses `user-groups` paths — document that for PM/support. Optional: cost‑only **i18n** overrides (`Messages.js` branches by deployment constant).

**Routes to extract (**`v2UserGroupsRouteElements` **— example decomposition)**

From `V2Routing` (users-and-user-groups segment):

1. **Shell + groups tab only** (if remote is “groups only”):

<Route {...v2Guard([principals.canList, groups.canView], { checkAll: false })}>

<Route path={pathnames['users-and-user-groups'].path} element={<UsersAndUserGroups />}>

<Route {...v2Guard([groups.canView])}>

<Route path={pathnames['user-groups'].path} element={<AccessManagementUserGroups />}>

<Route path={pathnames['create-user-group'].path} element={<AddGroupWizard />} />

</Route>

</Route>

</Route>

</Route>

<Route {...v2Guard([groups.canCreate])}>

<Route path={pathnames['users-and-user-groups-edit-group'].path} element={<EditUserGroup />} />

<Route path={pathnames['users-and-user-groups-create-group'].path} element={<CreateUserGroup />} />

</Route>

**Provider stack**


| **Provider**           | **Needed?**                                                                                            |
| ---------------------- | ------------------------------------------------------------------------------------------------------ |
| Full `Iam`‑class stack | **Yes** — groups use shared queries, V1 wizard paths, possibly inventory/cost delegates inside wizards |
| `AccessCheck.Provider` | **Yes**                                                                                                |


**Cost on‑prem**

- Remove **“add service accounts”** UI paths by **compile‑time** code branches + **excluding** service‑account–specific routes/components (grep `service-account`, `ServiceAccount`, `group-service-accounts` under `src/v1/features/groups` and any V2 group detail consumers).
- Keep **principal‑focused** group UX aligned with IDP sync story.

**Example URLs**

/iam/access-management/users-and-user-groups/user-groups

/iam/access-management/users-and-user-groups/user-groups/create-user-group

/iam/access-management/users-and-user-groups/edit-group/:groupId

---

## **4. SaaS vs Cost on‑prem builds (engineering spec)**


| **Dimension**           | **SaaS artifact**                                              | **Cost on‑prem artifact**                              |
| ----------------------- | -------------------------------------------------------------- | ------------------------------------------------------ |
| Build arg               | e.g. `RBAC_UI_DEPLOYMENT=saas`                                 | `RBAC_UI_DEPLOYMENT=cost-onprem`                       |
| Webpack                 | `DefinePlugin` injects literal into `src/shared/deployment.ts` | Same                                                   |
| `fec.config.js` exposes | Full set of agreed remotes                                     | Subset + possibly `IamCostOnPrem` only                 |
| `V2Routing`             | Full tree                                                      | `v2CostOnPremRouteElements` only (subset of fragments) |
| Overview                | Full shared Overview                                           | Stripped Overview                                      |
| Users                   | Invite route + UI                                              | No invite route + UI                                   |
| Groups                  | Full group/service‑account UX                                  | No add‑service‑accounts                                |


---

## **5. Optional: one “bundle” remote vs many**


| **Approach**                                                               | **Pros**                  | **Cons**                                        |
| -------------------------------------------------------------------------- | ------------------------- | ----------------------------------------------- |
| **Separate remotes** (`OverviewAccessManagement`, `MyAccessManagement`, …) | Host loads minimal JS     | Duplicated provider shells (acceptable if thin) |
| **Single** `IamCostOnPremShell` remote                                     | One `AsyncComponent` line | Larger chunk                                    |


Recommendation: **mirror roles** — **one remote per major slice** for SaaS flexibility; cost image may expose **only** the remotes Cost shell actually mounts.

---

## **6. Verification matrix (QA)**


| **Remote** | **Storybook story** | **MSW**                            | **Play function smoke** |
| ---------- | ------------------- | ---------------------------------- | ----------------------- |
| Overview   | Federated default   | Optional                           | H1 / key cards          |
| My Access  | Tab navigation      | Per MyGroups/MyWorkspaces handlers |                         |
| Users      | List + overflow     | users handlers                     | No invite in cost build |
| Roles      | Existing pattern    | roles handlers                     | Grid visible            |
| Groups     | Group list + create | groups handlers                    | No SA add in cost build |


---

## **7. Dependencies outside this repo**

- **On‑prem gateway:** `/api/rbac/...`, `/api/kessel/v1beta2` (or agreed path).
- **Cost shell:** chrome shim + Router `basename` + MF loader config for `rbac` remote URL.
- **Identity Providers page** (from earlier product notes): **not covered above** — add `v2IdentityProvidersRouteElements` + API client when backend contract exists.
- 



