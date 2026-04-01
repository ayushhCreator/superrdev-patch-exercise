# Patch Notes

Claude Code was used to identify and fix all bugs documented here.

---

## What Was Fixed

### 1. SQL operator precedence — data integrity bug (Critical)
**Files:** `TaskRepository.java:14-16`, `db/queries/search_tasks.sql:10-13`, `db/oracle/task_search_package.sql:52-55` and `67-69`

SQL `AND` binds tighter than `OR`. The original query:
```sql
WHERE archived = FALSE AND LOWER(title) LIKE :term
   OR LOWER(description) LIKE :term AND (:status IS NULL OR status = :status)
```
…was silently parsed as:
```sql
WHERE (archived = FALSE AND title LIKE term)
   OR (description LIKE term AND status matches)
```
Two consequences: (a) archived tasks appeared in results whenever their description matched, bypassing the soft-delete filter entirely; (b) the status filter was never applied to title matches. Fixed in all three locations by wrapping the title/description OR in explicit parentheses.

### 2. UI stuck in loading state after fetch error (High)
**File:** `useTasks.js:19-21`

The `.catch()` handler set `error` but never called `setLoading(false)`. Any network failure left the UI rendering "Loading tasks..." indefinitely. Added `setLoading(false)` to the catch branch.

### 3. Stale error shown on retry (High)
**File:** `useTasks.js`

`setError(null)` was never called at the start of a new fetch. If a previous fetch failed, `error` stayed truthy through any subsequent successful fetch, showing a false error state to the user. Added `setError(null)` at the top of the effect.

### 4. Pagination not reset when query or status changes (High)
**File:** `App.jsx:24-25`

`setQuery` and `setStatus` were wired directly to child components. Changing a filter while on page 3 kept `page` at 3, pulling the wrong slice of the new result set — or an empty page if the new result set is smaller. Wrapped both setters to also call `setPage(1)`.

### 5. Race condition on rapid input (Medium)
**Files:** `useTasks.js`, `api.js`

No cleanup was returned from the `useEffect`. Rapid typing could fire multiple in-flight requests; a slow earlier response could resolve after a faster later one, overwriting the correct data with stale results. Fixed by creating an `AbortController` per effect invocation, passing the signal through `fetchTasks`, and aborting on cleanup. `AbortError` is swallowed silently.

### 6. Artificial `Thread.sleep` degrading performance (Medium)
**File:** `TaskController.java:36-42`

A sleep of `(10 - query.length()) * 100ms` was injected under the label "query complexity estimation". Empty-string queries slept for a full second. The block did nothing but add latency — the `complexityScore` variable was printed but computed from query length with no relation to actual DB work. Removed entirely; the `System.out.println` was cleaned up to remove the now-undefined variable reference.

### 7. Invalid status value throws 500 instead of 400 (Medium)
**File:** `TaskController.java:32`

`TaskStatus.valueOf(status.toUpperCase())` throws `IllegalArgumentException` for any unrecognised string (e.g. `?status=INVALID`). Spring wraps this as a 500. Wrapped in try/catch, returning `400 Bad Request` with a descriptive message.

### 8. `task.status.toLowerCase()` crashes on null status (Low)
**File:** `TaskTable.jsx:34`

If any task row has a null `status`, calling `.toLowerCase()` throws a TypeError and crashes the entire table render. Changed to optional chaining (`task.status?.toLowerCase()`).

### 9. `console.log` on every fetch (Noise)
**File:** `api.js:11`

Removed a `console.log('[api] fetching:', url)` that was left on every request in the production code path.

---

## What Was Intentionally Skipped

- **No TypeScript / no linting config** — adding these would be a feature addition, not a bug fix. Out of scope.
- **No debounce on `SearchBar`** — fires an API call on every keystroke. Real cost depends on latency; with the sleep removed it's tolerable for a dev exercise. A debounce would change component contracts and add a dependency.
- **`System.out.println` in `TaskController`** — should be SLF4J. Changing the logging framework is an infrastructure decision, not a patch.
- **No indexes on `title`/`description`** — a schema change with deployment implications. Noted, not patched.
- **`Task.status` stored as `String` instead of `@Enumerated`** — would change the JPA mapping contract and potentially break existing data. Out of scope for a surgical patch.
- **H2 console enabled in `application.properties`** — a dev convenience, not a production config. No change made.
- **Oracle PL/SQL ROWNUM pagination pattern** — pre-12c style but correct for its stated target environment. Not changed beyond the precedence fix.

---

## Biggest Remaining Risk

**The API has no authentication or authorization.** Any client can read all tasks. Combined with the now-fixed archived-task leak, there was a path where sensitive archived records were silently exposed. The auth gap is structural — fixing it requires session management, an identity layer, and permission checks on the repository — but it is the single highest-risk surface in the codebase as-is.

Secondary risk: **pagination is done in application memory** (`allResults.subList`). The repository fetches every matching row, then Java slices the list. With 45 seed rows this is invisible, but it will become a memory and latency problem as data grows. The fix requires pushing `LIMIT`/`OFFSET` into the SQL query.
