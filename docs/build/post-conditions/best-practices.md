---
description: Common patterns, testing strategies, and gotchas for working with post-conditions in Stacks.
---

# Best Practices

This guide covers practical patterns for using post-conditions correctly, how to test them with the Clarinet SDK, and common mistakes to avoid. It assumes you are already familiar with [what post-conditions are](overview.md) and [how to construct them](implementation.md).

***

## Common patterns

### Pattern 1: Simple STX transfer

The most common pattern — a user sends a fixed amount of STX to a contract.

{% tabs %}
{% tab title="Clarity" %}
```clarity
(define-public (donate (amount uint))
  (begin
    (try! (stx-transfer? amount tx-sender (as-contract tx-sender)))
    (ok amount)
  )
)
```
{% endtab %}

{% tab title="stacks.js" %}
```typescript
import { Pc } from '@stacks/transactions';
import { request } from '@stacks/connect';

const postConditions = [
  Pc.principal(userAddress).willSendEq(amount).ustx()
];

await request('stx_callContract', {
  contractAddress,
  contractName: 'donation-tracker',
  functionName: 'donate',
  functionArgs: [Cl.uint(amount)],
  postConditions,
  postConditionMode: 'deny',
  network: 'mainnet',
});
```
{% endtab %}
{% endtabs %}

Use `.willSendEq()` when the transfer amount is exact and known ahead of time. This is the most restrictive and safest option.

***

### Pattern 2: Multiple transfers in one transaction

When a single contract function performs more than one transfer involving the same principal, you need a separate post-condition for each transfer.

{% tabs %}
{% tab title="Clarity" %}
```clarity
;; Splits the donation: half to the contract, half to a recipient
(define-public (split-donation (amount uint) (recipient principal))
  (let ((half (/ amount u2)))
    (try! (stx-transfer? half tx-sender (as-contract tx-sender)))
    (try! (stx-transfer? half tx-sender recipient))
    (ok half)
  )
)
```
{% endtab %}

{% tab title="stacks.js" %}
```typescript
import { Pc } from '@stacks/transactions';

const half = amount / 2n;

const postConditions = [
  // First transfer: user → contract
  Pc.principal(userAddress).willSendEq(half).ustx(),
  // Second transfer: user → recipient
  Pc.principal(userAddress).willSendEq(half).ustx(),
];

await request('stx_callContract', {
  contractAddress,
  contractName: 'donation-tracker',
  functionName: 'split-donation',
  functionArgs: [Cl.uint(amount), Cl.principal(recipientAddress)],
  postConditions,
  postConditionMode: 'deny',
  network: 'mainnet',
});
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
In `deny` mode, every transfer that occurs during execution must be covered by a post-condition. If you only declare one post-condition for a function that makes two transfers, the second uncovered transfer will cause the transaction to abort.
{% endhint %}

***

### Pattern 3: Conditional transfer

Some contract functions only transfer assets when a condition is met (for example, blocking repeat actions). If you are in `deny` mode and no transfer occurs, a post-condition that expects a transfer will cause the transaction to abort.

{% tabs %}
{% tab title="Clarity" %}
```clarity
(define-constant ERR_ALREADY_DONATED (err u102))
(define-map donations principal uint)

;; Only first-time donors can donate
(define-public (donate-once (amount uint))
  (begin
    (asserts! (is-none (map-get? donations tx-sender)) ERR_ALREADY_DONATED)
    (try! (stx-transfer? amount tx-sender (as-contract tx-sender)))
    (map-set donations tx-sender amount)
    (ok amount)
  )
)
```
{% endtab %}

{% tab title="stacks.js" %}
```typescript
import { Pc } from '@stacks/transactions';

// The transfer only happens for first-time donors.
// A post-condition with willSendEq is still declared — if the contract
// aborts (repeat donor), no transfer occurs and the post-condition is
// not evaluated against a failed transaction.
const postConditions = [
  Pc.principal(userAddress).willSendEq(amount).ustx()
];

await request('stx_callContract', {
  contractAddress,
  contractName: 'donation-tracker',
  functionName: 'donate-once',
  functionArgs: [Cl.uint(amount)],
  postConditions,
  postConditionMode: 'deny',
  network: 'mainnet',
});
```
{% endtab %}
{% endtabs %}

When a contract call fails and aborts (for example, `ERR_ALREADY_DONATED`), the transaction reverts and no asset transfers occur. Post-conditions are only evaluated against the asset movements that actually happen, so a post-condition for a transfer that never occurs will not cause an additional abort.

***

### Pattern 4: Read-only functions

Read-only functions do not transfer assets, so no post-conditions are needed. You do not need to call them with `postConditions` or a `postConditionMode`.

```typescript
// No post-conditions needed for read-only calls
const result = await fetchCallReadOnlyFunction({
  contractAddress,
  contractName: 'donation-tracker',
  functionName: 'get-total-raised',
  functionArgs: [],
  network: 'mainnet',
  senderAddress: userAddress,
});
```

***

## Testing post-conditions with Clarinet

The Clarinet SDK simulates the Stacks blockchain locally, so you can verify that your contracts behave correctly under the post-conditions you intend to declare on the client. The tests below use [Vitest](../clarinet/testing-with-clarinet-sdk.md).

### Setting up the test environment

```typescript
import { describe, expect, it } from 'vitest';
import { Cl } from '@stacks/transactions';

const accounts = simnet.getAccounts();
const deployer = accounts.get('deployer')!;
const alice = accounts.get('wallet_1')!;
const bob = accounts.get('wallet_2')!;
```

### Testing a simple transfer

```typescript
it('records donation and updates total', () => {
  const amount = 1_000_000; // 1 STX in uSTX

  const result = simnet.callPublicFn(
    'donation-tracker',
    'donate',
    [Cl.uint(amount)],
    alice
  );

  expect(result.result).toBeOk(Cl.uint(amount));

  const donation = simnet.callReadOnlyFn(
    'donation-tracker',
    'get-donation',
    [Cl.principal(alice)],
    alice
  );
  expect(donation.result).toBeOk(Cl.uint(amount));
});
```

### Testing a split transfer

```typescript
it('splits donation between contract and recipient', () => {
  const amount = 1_000_000;
  const half = amount / 2; // Clarity integer division rounds down

  const result = simnet.callPublicFn(
    'donation-tracker',
    'split-donation',
    [Cl.uint(amount), Cl.principal(bob)],
    alice
  );

  expect(result.result).toBeOk(Cl.uint(half));

  // Only the half sent to the contract is tracked
  const donation = simnet.callReadOnlyFn(
    'donation-tracker',
    'get-donation',
    [Cl.principal(alice)],
    alice
  );
  expect(donation.result).toBeOk(Cl.uint(half));
});
```

### Testing a conditional transfer

```typescript
it('allows first-time donation', () => {
  const result = simnet.callPublicFn(
    'donation-tracker',
    'donate-once',
    [Cl.uint(1_000_000)],
    alice
  );
  expect(result.result).toBeOk(Cl.uint(1_000_000));
});

it('rejects repeat donation', () => {
  simnet.callPublicFn('donation-tracker', 'donate-once', [Cl.uint(1_000_000)], alice);

  const result = simnet.callPublicFn(
    'donation-tracker',
    'donate-once',
    [Cl.uint(1_000_000)],
    alice
  );

  expect(result.result).toBeErr(Cl.uint(102)); // ERR_ALREADY_DONATED
});
```

{% hint style="info" %}
Simnet tests verify that your contract logic is correct but do not enforce client-side post-conditions. Client-side post-conditions are evaluated by the Stacks node when the signed transaction is broadcast. Use simnet tests to confirm the contract behaves as intended, then use integration tests or devnet to verify the full post-condition flow end to end.
{% endhint %}

***

## Common gotchas

### 1. Forgetting a post-condition for each transfer

When a function makes multiple transfers, each one must be covered by its own post-condition in `deny` mode.

{% tabs %}
{% tab title="Wrong" %}
```typescript
// Function makes two STX transfers, but only one post-condition is declared.
// The second transfer has no corresponding post-condition → transaction aborts.
const postConditions = [
  Pc.principal(userAddress).willSendEq(half).ustx(),
];
```
{% endtab %}

{% tab title="Correct" %}
```typescript
// One post-condition per transfer.
const postConditions = [
  Pc.principal(userAddress).willSendEq(half).ustx(),
  Pc.principal(userAddress).willSendEq(half).ustx(),
];
```
{% endtab %}
{% endtabs %}

***

### 2. Using `allow` mode instead of `deny`

`allow` mode lets the contract transfer any asset beyond your declared post-conditions. This removes the primary protection post-conditions provide.

{% tabs %}
{% tab title="Wrong" %}
```typescript
await request('stx_callContract', {
  // ...
  postConditions: [myCondition],
  postConditionMode: 'allow', // Contract can transfer anything else too
});
```
{% endtab %}

{% tab title="Correct" %}
```typescript
await request('stx_callContract', {
  // ...
  postConditions: [myCondition],
  postConditionMode: 'deny', // Only declared transfers are permitted
});
```
{% endtab %}
{% endtabs %}

Always default to `deny` mode. Use `allow` only when you have an explicit reason to permit transfers beyond your declared conditions, such as interacting with a contract that performs dynamic transfers you cannot enumerate ahead of time.

***

### 3. Token name mismatch

The token name in `.ft()` must exactly match the asset name defined in the contract, not the contract name.

{% tabs %}
{% tab title="Wrong" %}
```clarity
;; Contract defines the asset name as 'my-token'
(define-fungible-token my-token)
```

```typescript
// Passing the contract name instead of the asset name
Pc.principal(userAddress).willSendEq(100).ft('SP123.my-ft-contract', 'my-ft-contract');
```
{% endtab %}

{% tab title="Correct" %}
```typescript
// Passing the asset name from the contract definition
Pc.principal(userAddress).willSendEq(100).ft('SP123.my-ft-contract', 'my-token');
```
{% endtab %}
{% endtabs %}

***

### 4. Using `willSendLte` when you mean `willSendEq`

Using `willSendLte` allows the contract to transfer anything from 0 up to your specified amount, including 0. For fixed-amount transfers, use `willSendEq` so users see the exact expected amount in their wallet.

{% tabs %}
{% tab title="Wrong" %}
```typescript
// Allows 0–1,000,000 uSTX. The contract could transfer nothing and pass.
Pc.principal(userAddress).willSendLte(1_000_000).ustx();
```
{% endtab %}

{% tab title="Correct" %}
```typescript
// Requires exactly 1,000,000 uSTX to be transferred.
Pc.principal(userAddress).willSendEq(1_000_000).ustx();
```
{% endtab %}
{% endtabs %}

Use range conditions (`.willSendLte`, `.willSendGte`) only when the transfer amount is genuinely dynamic — for example, when it depends on blockchain state at execution time.

***

### 5. Post-conditions do not cover the receiving side

Post-conditions check who *sends* an asset and how much. They do not verify who receives it, or what an address holds after the transaction completes.

```typescript
// This confirms the user sends exactly 10 STX.
// It does NOT confirm the contract receives it, or that anyone else receives it.
Pc.principal(userAddress).willSendEq(10_000_000).ustx();
```

If you need to verify the destination of a transfer, you must audit the contract code directly or use additional on-chain checks inside the contract.

***

## Quick reference

| Scenario | Pattern | `postConditionMode` |
|---|---|---|
| Fixed STX transfer | `.willSendEq(amount).ustx()` | `deny` |
| Dynamic STX transfer | `.willSendGte(min).ustx()` + `.willSendLte(max).ustx()` | `deny` |
| Multiple transfers | One `Pc` per transfer | `deny` |
| Fungible token | `.willSendEq(amount).ft(contractId, tokenName)` | `deny` |
| NFT transfer | `.willSendAsset().nft(assetId, tokenId)` | `deny` |
| NFT protection | `.willNotSendAsset().nft(assetId, tokenId)` | `deny` |
| No asset transfers | Empty `postConditions: []` | `deny` |
| Read-only call | Not applicable | Not applicable |

***

### Additional Resources

* [Post-Conditions Overview](overview.md) — What post-conditions are and how they work
* [Implementation](implementation.md) — Full `Pc` API reference with all condition types
* [Examples](examples.md) — Additional worked examples for STX, tokens, and NFTs
* [Working examples repo](https://github.com/bastiatai/post-conditions-examples) — Runnable Clarinet project with tests for all patterns above
