# Good and Bad Tests

## Good Tests

**Integration-style**: Test through real interfaces, not mocks of internal parts.

```typescript
// GOOD: Tests observable behavior
test("user can checkout with valid cart", async () => {
  const cart = createCart();
  cart.add(product);
  const result = await checkout(cart, paymentMethod);
  expect(result.status).toBe("confirmed");
});
```

Characteristics:

- Tests behavior users/callers care about
- Uses public API only
- Survives internal refactors
- Describes WHAT, not HOW
- One logical assertion per test

## Bad Tests

**Implementation-detail tests**: Coupled to internal structure.

```typescript
// BAD: Tests implementation details
test("checkout calls paymentService.process", async () => {
  const mockPayment = jest.mock(paymentService);
  await checkout(cart, payment);
  expect(mockPayment.process).toHaveBeenCalledWith(cart.total);
});
```

Red flags:

- Mocking internal collaborators
- Testing private methods
- Asserting on call counts/order
- Test breaks when refactoring without behavior change
- Test name describes HOW not WHAT
- Verifying through external means instead of interface

```typescript
// BAD: Bypasses interface to verify
test("createUser saves to database", async () => {
  await createUser({ name: "Alice" });
  const row = await db.query("SELECT * FROM users WHERE name = ?", ["Alice"]);
  expect(row).toBeDefined();
});

// GOOD: Verifies through interface
test("createUser makes user retrievable", async () => {
  const user = await createUser({ name: "Alice" });
  const retrieved = await getUser(user.id);
  expect(retrieved.name).toBe("Alice");
});
```

**Tautological tests**: Restating the implementation back to itself. These provide zero confidence - they pass by construction and would still pass if the behavior were wrong.

**Do not add tests which simply restate the implementation.** A test must be able to fail for a real reason; if the only way it could fail is a typo in the test itself, delete it.

```typescript
// BAD: Restates the implementation. Mirrors the formula instead of
// pinning the behavior - if the formula is wrong, the test is wrong the same way.
test("applyDiscount subtracts the discount", () => {
  const price = 100;
  const discount = 0.2;
  expect(applyDiscount(price, discount)).toBe(price - price * discount);
});

// GOOD: Asserts a concrete, independently-known result
test("applyDiscount applies 20% off", () => {
  expect(applyDiscount(100, 0.2)).toBe(80);
});
```

Red flags for tautological tests:

- Computing the expected value with the same formula the code uses
- Asserting a constant equals the same constant the code returns
- Re-deriving the answer from inputs instead of stating the known answer
- A test that can only fail if you mistype the test
