# Code Review Checklist

Detailed prompts per review dimension. Read during step 4 (Analyze). Skip sections that do not apply to the code under review.

---

## Correctness

- Does the logic produce the intended result for typical inputs?
- Off-by-one errors in loops, slices, pagination, or array indices?
- Boolean conditions inverted or missing a branch?
- Type coercion surprises (loose equality, implicit string/number conversion)?
- Async/await: missing `await`, unhandled promise rejection, fire-and-forget side effects?
- State mutations: shared mutable state updated unexpectedly?
- Race conditions: read-then-write without locking or atomic update?
- Wrong operator precedence or missing parentheses changing behavior?
- Return values: early returns skipping cleanup, wrong default, unreachable code after return?

**When you see:** a `map`/`filter`/`reduce` chain — verify empty-array and single-element behavior.

---

## Errors & edge cases

- What happens on `null`, `undefined`, empty string, empty array, zero, negative numbers?
- Are errors caught and handled, or swallowed silently?
- Partial failure: if step 2 fails, is step 1 rolled back or left inconsistent?
- Network/IO timeouts and retries considered?
- User-facing error messages helpful without leaking internals?
- Validation at system boundaries (API input, file upload, query params)?
- Default values masking missing required data?

**When you see:** optional chaining (`?.`) — confirm it is intentional, not hiding a bug that should fail fast.

---

## Security

- User input interpolated into SQL, shell commands, HTML, or file paths?
- XSS: unescaped user content rendered in DOM or HTML templates?
- Auth: endpoints missing authentication or authorization checks?
- IDOR: resource access keyed only by predictable ID without ownership check?
- Secrets, API keys, or tokens hardcoded or logged?
- Unsafe deserialization of untrusted data?
- Path traversal in file operations (`../` in user-supplied paths)?
- CORS, CSP, or cookie flags misconfigured?
- Sensitive data in client-side code or error responses?

**When you see:** `dangerouslySetInnerHTML`, `eval`, `exec`, raw SQL string concat — treat as Critical until proven safe.

---

## Performance

- N+1 queries: fetching related records inside a loop?
- Unnecessary re-renders: missing memoization, unstable object/array literals in deps?
- Large data loaded entirely when pagination/streaming would suffice?
- Synchronous blocking I/O on a hot request path?
- Missing database indexes on filtered/sorted columns?
- Expensive computation inside render or tight loops?
- Memory leaks: event listeners, intervals, subscriptions not cleaned up?
- Unbounded caches or collections growing without eviction?

**When you see:** React `useEffect` with object/array deps — check for infinite re-fetch loops.

---

## API & contracts

- Public function signatures changed without updating callers?
- Return type inconsistent across code paths (sometimes object, sometimes null)?
- Breaking changes to REST/GraphQL schema, event payloads, or shared types?
- Leaky abstraction: internal implementation details exposed to consumers?
- Error response shape inconsistent with the rest of the API?
- Idempotency: retry-safe for mutations that should be?
- Versioning or deprecation path for removed endpoints/fields?

**When you see:** a shared type or interface change — trace at least one caller and one consumer.

---

## Best practices

- Does the code follow framework idioms (React hooks rules, Next.js server/client boundaries, Payload hooks)?
- Single responsibility: function/class doing too many unrelated things?
- Naming: variables and functions reveal intent without comments?
- Immutability where the codebase expects it?
- Configuration externalized (env vars) instead of hardcoded?
- Logging at appropriate levels; no debug noise in production paths?
- Dependencies: using a heavy library for something the stdlib/framework already provides?

**When you see:** a new pattern — compare with the nearest similar file in the project before flagging.

---

## Code smells

- **Long function** (>40 lines): extract logical blocks?
- **Deep nesting** (>3 levels): early returns or guard clauses?
- **God object/class**: too many responsibilities?
- **Feature envy**: method uses another object's data more than its own?
- **Shotgun surgery**: one change requires edits in many unrelated files?
- **Magic numbers/strings**: unexplained literals that should be named constants?
- **Boolean blindness**: `process(user, true, false, true)` — use options object or enum?
- **Primitive obsession**: passing strings/ints where a domain type would clarify?
- **Dead code**: unused imports, unreachable branches, commented-out blocks?

---

## Duplication

- Copy-pasted blocks differing only in variable names?
- Parallel implementations of the same algorithm in multiple files?
- Repeated conditional chains (`if type === 'a' ... else if type === 'b'`) — candidate for map/strategy?
- Similar validation logic duplicated across endpoints?
- Test setup duplicated across files — shared fixture or helper?

**Refactor signal:** three or more occurrences of the same pattern → suggest extraction with effort estimate.

---

## Testing

- New behavior covered by at least one test?
- Edge cases tested: empty, null, error path, boundary values?
- Tests assert behavior, not implementation details?
- Brittle tests: coupled to internal structure, timing, or random IDs?
- Mocking smell: mocking so much that the test verifies nothing real?
- Flaky patterns: `setTimeout` without wait, dependency on execution order?
- Deleted or skipped tests without explanation?

**When you see:** a bug fix with no test — flag as Major unless the area is explicitly untested by project convention.

---

## Maintainability

- Dead code, unused exports, or orphaned files?
- Unclear module boundaries or circular dependencies?
- Comments explaining *what* instead of *why* (or outdated comments)?
- Missing docs on non-obvious business rules or invariants?
- TODO/FIXME without ticket reference or owner?
- Over-engineering: abstraction layers with only one implementation?
- Under-engineering: hack with no note explaining the constraint?

---

## Stack-specific quick checks

### TypeScript / React

- `any` or unchecked type assertions hiding real type errors?
- Missing dependency array entries in `useEffect`/`useMemo`/`useCallback`?
- Server Component vs Client Component boundary respected (Next.js)?
- Keys on list items stable and unique?
- Props drilled deeply — context or composition alternative?

### Python

- Mutable default arguments (`def f(items=[])`)? 
- Bare `except:` swallowing all exceptions?
- Resource leaks: files/connections not used as context managers?
- Type hints on public APIs consistent with the codebase?

### SQL / database

- Parameterized queries (no string interpolation)?
- Migrations reversible or documented as one-way?
- Indexes on foreign keys and frequent filter columns?
- Transaction boundaries around multi-step writes?

### Payload CMS

- Access control on new collections/fields?
- Hook side effects idempotent and transaction-safe?
- Relationship depth and population performance?
- Validation on fields that accept user/admin input?
