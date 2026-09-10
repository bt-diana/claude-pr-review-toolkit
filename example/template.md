# Review Template: API Querying in React

**Task description:** https://github.com/rolling-scopes-school/tasks/blob/master/react/modules/tasks/queries.md  
**Branch:** `api-queries` (from `app-state-management`)  
**Max score:** 100 points

---

## PR Format Check

- [ ] Link to the task description
- [ ] Screenshot of the working application
- [ ] Deployment URL
- [ ] Submission date and deadline date
- [ ] Student's self-assessment with score breakdown
- [ ] No commented-out code, `node_modules`, or irrelevant files
- [ ] Commit history follows [RS School Git convention](https://rs.school/docs/git-convention)

---

## Overall feedback

<!-- 1–2 sentences of genuine positive feedback. 1–2 sentences on the main area for improvement. -->

---

## Code Quality

### Repository & Git (3 pts)

- [ ] **(1 pts)** Branch is created from `app-state-management` and named `api-queries`
- [ ] **(1 pts)** Commit history is meaningful and reflects the development process
- [ ] **(1 pts)** Commits follow the RS School naming convention

**Comment:**

---

### TypeScript (6 pts)

- [ ] **(2 pts)** `any` type is not used anywhere
- [ ] **(4 pts)** Enums, Generics, Object Types, and Function Types used appropriately — prefer literal unions over bare `string` for constrained fields

**Comment:**

---

### Architecture & Structure (15 pts)

- [ ] **(5 pts)** Code is divided into logical layers (query definitions, components, hooks, utilities) — files without JSX use the `.ts` extension, not `.tsx`
- [ ] **(10 pts)** No "god" components — each component has a single, clear responsibility

**Comment:**

---

### API & Query Layer (20 pts)

- [ ] **(4 pts)** RTK Query (Redux projects) or TanStack Query (Zustand projects) is used — no raw `fetch`/`axios` outside of query definitions
- [ ] **(4 pts)** Every API call is migrated to the query library — no ad-hoc client-side fetches remain
- [ ] **(4 pts)** Cache TTL is read from an environment variable, not hardcoded (e.g. `keepUnusedDataFor` must not be a literal)
- [ ] **(4 pts)** Cache invalidation is handled explicitly (e.g. `invalidatesTags` in RTK Query or `queryClient.invalidateQueries` in TanStack Query)
- [ ] **(4 pts)** Redux users: RTK Query hooks used directly in components and data/status read from cache — not mirrored into component state; Zustand users: custom hooks wrap TanStack queries

**Comment:**

---

### Code Quality Principles (30 pts)

- [ ] **(6 pts)** DRY — no duplicate code (e.g. repeated inline `useSelector` shapes should be a shared typed selector)
- [ ] **(6 pts)** KISS — solutions are simple and readable (no ternaries returning `true`/`false`; use the boolean expression directly)
- [ ] **(6 pts)** YAGNI — no dead code, unused variables, unused exports, or leftover dev-only controls
- [ ] **(5 pts)** No magic numbers or strings (e.g. API base URL extracted to a named constant)
- [ ] **(5 pts)** Variable, function, and component names are clear and descriptive — component name matches its file name; a state and its setter use consistent wording
- [ ] **(2 pts)** No commented-out code

**Comment:**

---

### Tooling (4 pts)

- [ ] **(2 pts)** ESLint configured for TypeScript; `no-explicit-any` enabled; zero errors and warnings — rules such as `react-hooks/exhaustive-deps` and `react-refresh/only-export-components` are not disabled to hide reports
- [ ] **(2 pts)** Prettier configured and applied consistently (line-ending/CRLF issues are not scored — they go under Additional recommendations)
### Tests (22 pts)

- [ ] **(10 pts)** Test coverage is ≥ 80%
- [ ] **(4 pts)** All tests pass
- [ ] **(8 pts)** Tests specifically cover the query layer: loading states, error states, and caching behaviour — tests exercise the app's own query service (not the RTK library primitives), every test has real assertions, and shared render/setup is extracted

**Comment:**

---

## Total: /100

_Preliminary score (pending fixes): /100_

---

## Additional recommendations

<!-- Non-scoring suggestions. State clearly that these don't affect the mark. -->
