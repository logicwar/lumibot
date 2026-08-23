# IBKR Integration Roadmap

## Purpose

Maintain a small, upstream-compatible Lumibot patch stack for Interactive
Brokers.

The project has two related goals:

1. Preserve and test the IB Gateway behaviour required by existing strategies.
2. Complete the IBKR Client Portal REST implementation sufficiently for a safe
   future migration from Gateway.

This repository must remain generic Lumibot code. It must not contain
PhoenixBot-specific strategy logic, credentials, account identifiers, local
paths, or operational configuration.

## Baseline

Initial development baseline:

- Upstream Lumibot release: `v4.5.83`
- Working branch: `ibkr`
- Official source remote: `Lumiwealth/lumibot`
- Custom changes: small, independently reviewable commits on top of upstream

A validated custom release will always be identified by an immutable tag, for
example:

```text
logicwar-v4.5.83-ibkr.1
```

## Current findings

### IB Gateway

The upstream Gateway implementation does not currently preserve several  
behaviours required for reliable advanced equity orders:

- Bracket take-profit and stop-loss child prices must use the order's actual  
  take-profit and stop-loss fields.
- Parent and child legs must consistently receive time-in-force settings.
- Good-Till-Date must be applied and formatted correctly for every applicable  
  bracket leg.
- The last-price fallback must use the actual cached price state, rather than  
  an unrelated or obsolete tick-state check.
- Backtesting must reproduce the intended bracket, TIF, and GTD semantics.

These behaviours exist as local patches in the PhoenixBot Lumibot snapshot and  
must be independently reproduced, documented, and regression-tested here.

### IBKR Client Portal REST

The current REST route does not fully implement advanced order behaviour:

- Bracket child orders are not serialized into an IBKR REST submission.
- Parent/child relationships and OCO exit semantics are incomplete.
- TIF and Good-Till-Date support must be verified for every order leg.
- Returned broker orders must be reconstructed into Lumibot parent/child  
  orders.
- Refresh, cancellation, partial fills, broker restart, and rejected-order  
  handling need explicit tests.

REST must not be considered a replacement for Gateway until its complete order  
lifecycle has been validated.

## Scope

### Phase 1 — Establish IB Gateway parity

Recreate the required Gateway fixes on the v4.5.83 baseline.

- Correct bracket parent, take-profit, and stop-loss order construction.
- Apply TIF and GTD consistently to parent and child legs.
- Correct the last-price fallback condition.
- Verify equivalent backtesting behaviour where Lumibot simulates these order  
  properties.
- Add focused regression tests for each corrected behaviour.

Definition of done:

- The known Gateway behaviours are covered by automated tests.
- The implementation is generic and contains no PhoenixBot code.
- A concise compatibility note explains any difference from upstream.
- The changes are suitable for an upstream pull request.

### Phase 2 — Define REST advanced-order contract

Before implementing REST submission, document the intended Lumibot-to-IBKR  
mapping.

- Confirm the current IBKR Client Portal API contract from official IBKR  
  documentation and controlled paper-account observations.
- Define how Lumibot `Order`, `OrderClass`, child orders, TIF, and GTD map to  
  REST request payloads.
- Define bracket entry, take-profit exit, stop-loss exit, OCO relationship,  
  cancellation, and replace semantics.
- Record unsupported order types explicitly rather than silently degrading  
  them to a single entry order.

Definition of done:

- A request/response mapping is reviewed and documented.
- Unsupported cases fail clearly before submission.
- No real-order submission occurs merely to discover payload shapes.

### Phase 3 — Implement REST bracket submission

Implement one safe advanced-order path at a time.

Initial target:

- Stock/ETF bracket order with one entry, one take-profit exit, and one  
  stop-loss exit.
- Supported entry types and exit types defined explicitly.
- TIF and GTD propagated to every valid leg.
- Stable Lumibot identifiers retained for parent and children.

Requirements:

- Never submit only the entry leg while presenting the result as a bracket.
- Validate required prices, quantities, sides, and child relationships before  
  making a REST request.
- Preserve sanitized request/response diagnostics without logging credentials  
  or account information.
- Add mocked request-payload and response-parsing tests.

### Phase 4 — REST state and lifecycle reconciliation

Make REST order state safe after normal and abnormal events.

- Parse broker-returned parent and child orders.
- Refresh open, filled, cancelled, rejected, and partially filled orders.
- Reconcile strategy state after a process restart.
- Handle cancellation of a parent and cancellation of individual children.
- Verify that an executed exit cancels or retires its sibling according to  
  actual IBKR behaviour.
- Make failures visible; do not fabricate successful order state.

### Phase 5 — Controlled paper validation

Only after automated coverage is complete, validate against an IBKR paper  
account.

- Use deliberately non-marketable test orders first.
- Test submit, read, cancel, reject, partial-fill handling where practical,  
  and restart reconciliation.
- Record only sanitized evidence and reproducible test steps.
- Do not use live trading for development validation.

Paper validation requires explicit operator authorization and must never be run  
as a default test.

### Phase 6 — Upstream contribution and release management

For every complete generic capability:

- Open a focused pull request against `Lumiwealth/lumibot`.
- Keep implementation commits small and independent.
- Retain the patch in this fork unless and until upstream has merged and  
  released an equivalent change.
- Rebase the `ibkr` branch onto selected official release tags.
- Re-run the relevant tests after every rebase.
- Tag every validated fork release.

## Non-goals

This project does not:

- modify PhoenixBot directly;
- replace the official Lumibot project wholesale;
- add strategy-specific execution logic;
- add credentials, Docker secrets, account details, or local machine setup;
- claim REST/Gateway parity before automated and paper validation;
- use live broker orders as routine tests.

## Validation levels

| Level                  | Purpose                                                          | Broker interaction            |
| ---------------------- | ---------------------------------------------------------------- | ----------------------------- |
| Unit                   | Payload construction, parsing, validation, state transitions     | None                          |
| Integration with fakes | REST transport and broker lifecycle behaviour                    | None                          |
| Paper validation       | Confirm real IBKR REST/Gateway behaviour                         | Paper only, explicit approval |
| PhoenixBot integration | Validate a pinned fork release through PhoenixBot's opt-in route | Separate future project       |

## Upstream synchronization policy

- Monitor upstream `dev` for upcoming changes and conflicts.
- Build and release only from named official Lumibot tags.
- Rebase the custom patch stack only after reviewing the upstream release.
- Use `--force-with-lease` only for the personal `ibkr` working branch.
- Never make PhoenixBot depend on an untagged branch or on upstream `dev`.


