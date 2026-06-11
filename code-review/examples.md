# Code Review Examples

Compact examples of well-formed findings at different severity levels.

---

## Example 1: Critical — null dereference

**Input (pasted snippet):**

```typescript
function getUserEmail(users: User[], id: string) {
  const user = users.find((u) => u.id === id)
  return user.email.toLowerCase()
}
```

**Finding:**

```markdown
### Critical
- **A** — **`getUserEmail` — null dereference when user not found.** `Array.find` returns `undefined` if no match; accessing `.email` throws at runtime. **Fix:** guard before access:

\`\`\`typescript
const user = users.find((u) => u.id === id)
if (!user?.email) return null
return user.email.toLowerCase()
\`\`\`
```

---

## Example 2: Major — duplication worth extracting

**Input (two similar blocks in one file):**

```typescript
// Block A
if (order.status === 'pending') {
  await sendEmail(order.userId, 'order-pending', { orderId: order.id })
  await logEvent('email_sent', { type: 'order-pending', orderId: order.id })
}

// Block B (20 lines later)
if (order.status === 'shipped') {
  await sendEmail(order.userId, 'order-shipped', { orderId: order.id })
  await logEvent('email_sent', { type: 'order-shipped', orderId: order.id })
}
```

**Findings:**

```markdown
### Major
- **A** — **`handleOrderUpdate` — duplicated notify-and-log pattern.** Two blocks differ only in status string and template name; a third status will likely copy-paste again. **Fix:** extract `notifyOrderStatus(order, status, template)`.

## Refactoring opportunities

| Opportunity | Benefit | Effort |
|-------------|---------|--------|
| Extract `notifyOrderStatus(order, status, template)` | Single place for email + logging; easier to add retry or metrics | low |
```

---

## Example 3: Suggestion — naming clarity

**Input:**

```python
def calc(d, t):
    return d * 0.2 if t == 1 else d * 0.1
```

**Finding:**

```markdown
### Suggestions
- **A** — **`calc` — unclear parameter names reduce readability.** `d` and `t` require reading the body to understand. Names like `discount` and `is_premium_customer` (or an enum) would match project naming elsewhere. **Fix:** rename parameters; no behavior change.
```

---

## Example 4: Full mini-report

**Input:** A 30-line API handler diff adding a new POST endpoint.

```markdown
# Code Review: POST /api/orders handler (diff)

## Summary
The endpoint structure matches existing handlers and validation is present. One Major issue: missing authorization check compared to sibling endpoints. Safe to merge after auth is added.

## Findings

### Major
- **A** — **`src/routes/orders.ts:42`** — No ownership check on `userId` param. Other routes in this file call `requireOwner(req, userId)` before mutating. **Fix:** add `requireOwner(req, order.userId)` before `createOrder`.

### Minor
- **B** — **`src/routes/orders.ts:58`** — Magic number `5000` for timeout. **Fix:** use `ORDER_TIMEOUT_MS` constant like `src/routes/payments.ts:12`.

### Suggestions
- **C** — **`src/routes/orders.ts:30-55`** — Handler mixes validation, DB write, and email dispatch. **Fix:** consider extracting `dispatchOrderConfirmation` when a third notification type is added.

## Refactoring opportunities

| Opportunity | Benefit | Effort |
|-------------|---------|--------|
| Extract notification dispatch from handler | Aligns with payments route pattern; easier to test | medium |

## Positive notes
- Input validation mirrors the Zod schema used in `payments.ts`
- Error responses use the shared `ApiError` shape consistently

## Open questions
- Is `userId` always derived from the session, or can it be supplied by the client? If client-supplied, severity is Critical (IDOR).
```
