# Review Template: API Querying in React

**Task description:** https://github.com/rolling-scopes-school/tasks/blob/master/react/modules/tasks/queries.md
**Branch:** `api-queries` (from `app-state-management`)
**Max score:** 100 points

## Overall feedback

## Code Quality

### Repository & Git (1/3 pts)

- [x] **(1/1 pts)** Branch is created from `app-state-management` and named `api-queries`

- [ ] **(0/1 pts)** Commit history is meaningful and reflects the development process

- [ ] **(0/1 pts)** Commits follow the RS School naming convention

**Comments:**

- 25db5d5 — "refactor: remove duplicated navigation logic using query helper" - exact duplicate of the previous commit's message; this commit only fixes indentation in DetailPanel/MainPage/navigation.test.
- d68c942 — "refactor: change constants name" - actually adds the whole refresh feature (handleRefresh, invalidateTags dispatch, Refresh button), fixes a POCKEMON→POKEMON typo, and deletes a stray test_output.txt file.
- df23bf9 — "feat: add refresh button" - the refresh button was already added in the previous commit; this one only adds CSS and reorders an import.
- 803e4be — "fix: refactor: use isFetching for pagination loading state" - uses two Conventional Commits type prefixes at once, which is not valid.
- f61b314 — "refactor: delete unnecessary ico file" - the diff actually deletes vite.svg, an SVG file, not an .ico file.
- 74e4f16 — "feat: update test for MainPage" - wrong type; the commit only changes whitespace in a test file, so it should use test:.
- 91119df — "feat: update tests" - undersells the change: it deletes the legacy API service, adds pokemonApi.ts, and adds the React plugin to the ESLint config.

### TypeScript (6/6 pts)

- [x] **(2/2 pts)** `any` type is not used anywhere

- [x] **(4/4 pts)** Enums, Generics, Object Types, and Function Types used appropriately — prefer literal unions over bare `string` for constrained fields

**Comments** (carried over from the earlier reviews — not scored):

- `tsconfig.app.json` and `tsconfig.node.json` are both missing `"strict": true` and `"noImplicitAny": true`.

### Architecture & Structure (15/15 pts)

- [x] **(5/5 pts)** Code is divided into logical layers (query definitions, components, hooks, utilities)
    
    **Comment**: Several files without JSX still use the `.tsx` extension instead of `.ts` (carried over from the earlier reviews — not scored).

- [x] **(10/10 pts)** No "god" components — each component has a single, clear responsibility

    **Comments:**

    - The app's `package.json` and `src` live in a nested `rs-react-app/` folder instead of the git root. This is a single frontend app, not a monorepo, so the project root should match the git root.

### API & Query Layer (19/20 pts)

- [x] **(4/4 pts)** RTK Query is used — no raw `fetch`/`axios` outside of query definitions

- [x] **(4/4 pts)** Every API call is migrated to the query library

- [ ] **(3/4 pts)** Cache TTL is read from an environment variable, not hardcoded

**Comment**: No `.env.example` documents the `VITE_CACHE_TTL` variable that `pokemonApi.ts` reads for `keepUnusedDataFor`.

- [x] **(4/4 pts)** Cache invalidation is handled explicitly

- [x] **(4/4 pts)** RTK Query hooks used directly in components and data/status read from cache — not mirrored into component state


### Code Quality Principles (29/30 pts)

- [ ] **(5/6 pts)** DRY — no duplicate code

    **Comment:** `createSearchQueryString({ page, search: searchFromUrl })` is built twice with the same arguments in `MainPage.tsx` — once for the outside-click handler

- [x] **(6/6 pts)** KISS — solutions are simple and readable

    **Comment:** the outside-click check in `MainPage.tsx` uses `(e.target as HTMLElement).closest('.pokemon-card')` (carried over from the earlier reviews — not scored). Calling `e.stopPropagation()` in the card itself, the way `ResultItem` already does for its checkbox, is simpler and more robust.

- [x] **(6/6 pts)** YAGNI — no dead code, unused variables, unused exports, or leftover dev-only controls

- [x] **(5/5 pts)** No magic numbers or strings

- [x] **(5/5 pts)** Variable, function, and component names are clear and descriptive

    **Comment:** `useSearchTermLocalStorage.test.tsx` still imports the hook under its old name (`useLocalStorage`) and groups the tests under a generic `describe('storage service', ...)` label (already flagged in the state-management review — not scored).

- [x] **(2/2 pts)** No commented-out code

### Tooling (4/4 pts)

- [x] **(2/2 pts)** ESLint configured for TypeScript; `no-explicit-any` enabled; zero errors and warnings

- [x] **(2/2 pts)** Prettier configured and applied consistently

### Tests (19/22 pts)

- [x] **(10/10 pts)** Test coverage is ≥ 80%

- [x] **(4/4 pts)** All tests pass

- [ ] **(5/8 pts)** Tests specifically cover the query layer: loading states, error states, and caching behaviour

    **Comments:**

    - `pokemonApi.test.ts` only covers success responses. It does not directly test the query layer's error path (fetch rejects / non-2xx) or caching behaviour (for example dispatching the same query twice and reusing the cache). Error and loading states are only exercised indirectly through UI mocks in `MainPage`/`DetailPanel` tests.
    - Test names in `pokemonApi.test.ts` and `Flyout.test.tsx` describe the scenario but do not start with "should", unlike the rest of the suite.
    - `useSearchTermLocalStorage.test.tsx` groups its tests under a generic `describe('storage service', ...)` instead of naming it after the hook (already counted under the naming criterion — not scored twice; +1 pt returned after a re-review).

## Total: 93/100

## Additional recommendations

- Component prop types are not wrapped in `Readonly<...>` (SonarQube `typescript:S6759`). React props are immutable by contract, so wrapping each props type in `Readonly<...>` prevents accidental mutation. Affected: `SearchProps` in `Search.tsx`.

- Lint fails repo-wide because of CRLF line endings (57 files report unformatted); there is no `.gitattributes` to normalize line endings. Add one with `* text=auto eol=lf` and run the formatter.
