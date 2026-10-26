# CLAUDE.md

React admin UI for Invoice Ninja. It is a pure SPA that talks to the Laravel API
(the backend lives in a separate repo, checked out locally at
`../../invoiceninja/invoiceninja`). Default branch for work is `develop`; PRs
target `develop` (releases go to `main`).

## Commands

```sh
npm run dev          # Vite dev server on http://localhost:3000
npm run build        # tsc --noEmit + vite build in parallel (this is the type check)
npm run lint         # biome lint src/
npm run format       # biome format --write src/
npm run test         # vitest, only tests/unit/**/*.test.ts
npx vitest run tests/unit/helpers.test.ts   # single unit test file
npm run test:e2e     # Playwright, serial (needs a seeded backend, see tests/e2e-guide.md)
npx playwright test tests/e2e/clients.spec.ts --project=chromium --workers=1
```

- Node >= 22 (`engineStrict`).
- The husky pre-commit hook runs `npm run lint && npm run build`, so commits
  fail on lint or type errors.
- CI (`.github/workflows/analyse.yml`) runs build, test and lint, and checks
  that the self-hosted bundle contains no Sentry code.
- `test:e2e:parallel` / `test:e2e:isolated` reference
  `scripts/playwright-spec-orchestrator.mjs`, which is not currently in the
  repo.

## Configuration

Env vars come from `.env` (copy `.env.example`). The important ones:

- `VITE_API_URL`: backend origin for self-hosted. `apiEndpoint()` in
  `src/common/helpers.ts` resolves it.
- `VITE_IS_HOSTED`: `'true'` switches to hosted (SaaS) behaviour. Use
  `isHosted()` / `isSelfHosted()` from `$app/common/helpers` rather than reading
  the env var directly.
- `VITE_ROUTER`: `browser` or `hash`.
- Many `VITE_ENABLE_*` / `VITE_DISABLE_*` flags gate features.
- Sentry is only for hosted builds (`sentry.config.ts`, `src/common/sentry.ts`).

## Architecture

### Path aliases

`$app/*` maps to `src/*` and `$tests/*` to `tests/*` (tsconfig `paths`). Use
`$app/...` imports for anything outside the current feature folder.

### Layout of `src/`

- `index.tsx`: bootstrap. Sets up i18next (English bundled), the React Query
  client, the Redux store, the router and Google OAuth. Also exports the `mitt`
  `emitter`.
- `App.tsx`: global side effects (locale loading, dark mode, socket events,
  keyboard shortcuts, session timeout) and renders `routes`.
- `common/`: shared, non-visual code.
  - `routes.tsx`: top-level route tree. It composes each page module's
    `routes.tsx`.
  - `queries/`: one file per API resource with React Query hooks
    (`useXQuery`, `useBlankXQuery`) and plain mutation functions (`bulk`, ...).
  - `interfaces/`: TypeScript types for API entities (`Invoice`, `Client`, ...).
  - `hooks/`: shared hooks (`useCurrentCompany`, `useCurrentUser`,
    `useHasPermission`, `useRefetch`, `useTitle`, ...).
  - `helpers/`: `request.ts` (axios wrapper), `toast/`, `route.ts`, dates,
    money and number formatting.
  - `guards/`: route guards (`permission`, `admin`, `assigned`, `enabled`,
    `plan`, `or`, ...).
  - `stores/`: Redux Toolkit store. Slices hold session state: `companyUsers`
    (the logged-in account/companies), `user`, `settings`, `products`,
    `companyDocuments`.
  - `atoms/`, `constants/`, `enums/`, `dtos/`.
- `components/`: shared UI. `forms/` (InputField, SelectField, Button, Toggle,
  Combobox, ...), `cards/`, `layouts/` (`Default` page shell, `Settings`
  shell), `DataTable.tsx`, `Modal.tsx`, plus domain components (`invoices/`,
  `clients/`, ...).
- `pages/<entity>/`: one folder per feature. A typical folder has
  - `routes.tsx` exporting e.g. `productRoutes`, with lazy-loaded pages wrapped
    in `<Guard guards={[...]} component={...} />`,
  - `index/`, `create/`, `edit/`, `show/`, `import/`, `documents/`
    subpages,
  - `common/` for feature-local components, hooks, atoms and queries,
  - often a parent component (e.g. `Product.tsx`) that renders the `Default`
    layout, owns the entity state and save handler, and passes them to
    subpages through `<Outlet context={...} />` / `useOutletContext()`.
- `resources/lang/<locale>/<locale>.json`: translations (45 locales).
  `resources/css/app.css`: Tailwind entry.

### State management

Several libraries are used side by side. Follow what the code you are editing
already uses.

- **Server state**: TanStack React Query. Query keys are the API path, e.g.
  `['/api/v1/products', id]`. Most queries use `staleTime: Infinity`, so after
  a mutation you must invalidate with `$refetch(['products'])` (or the
  `useRefetch()` hook). The key map and its cross-entity dependencies live in
  `src/common/hooks/useRefetch.tsx`. Add new resources there.
- **Session/global state**: Redux Toolkit (`useSelector`, the
  `useCurrentCompany` / `useCurrentUser` hooks).
- **Local shared UI state**: Jotai atoms (most common), some Zustand.
- **Events**: `window` `CustomEvent`s (`refetch`, `display.error.toaster`,
  `reset.password.required`) and the `mitt` `emitter`.

### API calls

Always go through `request(method, url, data?, config?)` from
`$app/common/helpers/request`, with the URL built by
`endpoint('/api/v1/products/:id', { id })` from `$app/common/helpers`. The
wrapper adds auth headers (`X-Api-Token`, `X-React`) and a response
interceptor that already handles global errors: toasts for 403/404/5xx, reload
on 401/429, and refetching activities after writes. A 422 rejects with a
`ValidationBag`. Pages store it in an `errors` state and pass it to form
fields. Pass `skipIntercept: true` to bypass the interceptor.

Typical save flow (see `src/pages/products/Product.tsx`):

```ts
toast.processing();
request('PUT', endpoint('/api/v1/products/:id', { id }), product)
  .then(() => {
    toast.success('updated_product');
    $refetch(['products']);
  })
  .catch((error: AxiosError<ValidationBag>) => {
    if (error.response?.status === 422) {
      setErrors(error.response.data);
      toast.dismiss();
    }
  });
```

### Permissions and plans

- Gate routes with `Guard` plus guard functions, and gate UI with
  `useHasPermission()('edit_invoice')`, `useAdmin()`, `useEntityAssigned`.
- Hosted plan checks (free, pro, enterprise) use the `plan` guards and
  `src/common/helpers/account-plan.ts`. Remember that self-hosted has no plan
  restrictions.

### UI conventions

- Styling uses Tailwind classes plus the theme palette from `useColorScheme()`
  (`src/common/colors.ts`). Colors are numbered tokens (`colors.$1`,
  `colors.$24`, ...) applied via `style={{ ... }}` so dark mode works. Don't
  hard-code colors.
- Some components use antd, Headless UI and styled-components.
- Icons come from `react-icons` / `react-feather` and `src/components/icons/`.
- User-facing text always goes through `const [t] = useTranslation(); t('key')`.
  `toast.success/error('key')` also takes translation keys. Translation keys
  are snake_case and mirror the backend's `lang/en/texts.php`. New strings go
  into `src/resources/lang/en/en.json`. The other locales are synced in bulk
  ("Updated translations" commits), so don't hand-edit them.
- Use `useTitle('translation_key')` to set the document title.

## Code style

- Biome: 2-space indent, single quotes, semicolons, ES5 trailing commas,
  80-column lines, organized imports. Run `npm run format` / `npm run check`.
- Every source file starts with the Invoice Ninja license header block
  (`@copyright ... @license https://www.elastic.co/licensing/elastic-license`).
  Copy it from a neighbouring file.
- Function components with named exports. Route-level pages that are
  lazy-loaded use `export default function`.
- Strict TypeScript. `any` is tolerated by the linter but prefer the
  interfaces in `common/interfaces`.

## Tests

- Unit tests live in `tests/unit/**`. Only `*.test.ts` files are picked up
  (no `.tsx`). They mirror the `src` structure (`helpers/`, `hooks/`,
  `pages/`, `queries/`).
- E2E tests live in `tests/e2e/*.spec.ts` and run against a production build
  (`vite build --mode testing`) served on port 4173 plus a real, seeded
  Laravel backend. Config comes from `.env.testing`. Tests share backend
  state, so keep `--workers=1` for direct runs. Read `tests/e2e-guide.md`
  before writing or running e2e tests.
