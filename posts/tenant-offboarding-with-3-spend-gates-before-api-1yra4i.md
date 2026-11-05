# Tenant Offboarding With 3 Spend Gates Before API Key Revocation and Data Erasure

Close workload admission and revoke tenant API keys before deleting user data; authorize cleanup through a separate service identity. For a B2B SaaS workload that must stop spending before the invoice arrives, deletion is the last destructive step, after already admitted work has reached a known outcome. The order is admission closure, credential revocation, reconciliation, then erasure.

TL;DR: Choose a spend ceiling only if the system can refuse new work and reserve a defensible upper bound for work it accepts. Offboarding should use that same admission boundary, while preserving enough control state to finish revocation and account for outstanding obligations. A deleted user row is poor evidence that access has ended.

This note proposes an architecture decision, with an illustrative budget ledger rather than measured production results. The security basis is the OWASP Secrets Management Cheat Sheet, which treats revocation, expiration, authorization, and secret lifecycle management as explicit concerns. The transaction boundaries and budget policy below are design requirements to implement and verify, rather than guarantees supplied by that guidance.

## Decision and the 3 admission gates

Adopt three logical gates: the tenant lifecycle gate, the credential authorization gate, and the workload reservation gate. A request must pass all three before it creates a billable obligation. They can share one authoritative transaction boundary; calling them separate gates doesn't require three services. For offboarding, make the lifecycle state terminal for ordinary admission, revoke the tenant's credentials, and run deletion through a narrowly scoped cleanup authority that does not depend on those credentials.

Refusal is intentional.

That choice resolves the central trade-off: if the business promises a ceiling, the admission path must be allowed to reject traffic when the remaining allocation cannot cover the next operation. An alert delivered after execution cannot satisfy that policy. Conversely, a team that values accepting every request above enforcing a ceiling should describe its policy as monitoring, and keep that distinction visible in its customer contract.

Consider an illustrative allocation of 1,000 accounting units for one workload, with 730 settled and 220 reserved. There are 50 units available for another reservation. A request whose defensible upper bound is 80 units must be refused, even if its expected consumption is lower. These are synthetic ledger values, not prices or observations. The difference between an expectation and an upper bound matters because a hard ceiling is only as credible as the obligations counted against it. Now let two workers each attempt a 40-unit reservation against that same remainder: the admission transaction must accept at most one, leaving 10 units available, even if both workers previously read a balance of 50. Reading the balance and writing a reservation in independent steps fails this example. The lifecycle check must participate in the same serialization, since otherwise a worker can observe an open tenant, pause while offboarding commits, and then reserve capacity against a tenant that has already closed.

During offboarding, refuse even the request that would fit. The allocation controls spend; lifecycle closure controls whether that tenant may create any new obligation at all.

## What must survive API key revocation and user data deletion?

The control record must survive long enough to establish a terminal tenant state, recognize retries, reconcile reservations, and record the outcome of cleanup. Keep it outside the user-data cascade, with an explicit retention and minimization policy. This is an architectural boundary, not permission to retain arbitrary personal data: a minimal tenant identifier, lifecycle version, operation identifier, and disposition can serve different purposes from the payload being erased, and their actual retention needs review against the application's obligations.

The admission invariant is precise: after lifecycle closure commits, no admission transaction for that tenant may commit a new reservation. Credential revocation must also prevent future use of the revoked credentials at every authorization boundary in scope. Those are related requirements, but neither settles work that was accepted earlier. A queued job, a running request, or an external operation may already represent an obligation; classify it as canceled with confirmation, completed and settled, or still unresolved before releasing its reservation.

Keep unresolved reservations.

Cancellation needs confirmation.

The budget invariant is settled consumption plus outstanding reservations no greater than the allocation. It depends on each reservation bounding the complete obligation, including any retries or downstream work allowed under that reservation. If an operation has no enforceable maximum, this design cannot claim a strict spend ceiling for it. Bound the operation, isolate it under an explicit exception, or refuse it. An invoice can also contain charges outside this workload ledger; define that accounting scope before promising a monetary cap.

Name the failure boundaries during review. An authorization cache that continues accepting a revoked key is a revocation propagation gap. A queue consumer that trusts an old authorization decision is a stale admission path. A timeout followed by an untracked retry is a duplicate obligation. A cascading deletion that removes the only reconciliation record is control-state loss. Each failure has a different correction, and none is repaired merely by changing which database delete statement runs first.

For a SaaS backend implemented in Node.js, the ordering is the same. The Python example below expresses the policy independently of an application runtime; each request handler, scheduler, and worker needs an integration point at the authoritative admission boundary.

## Options and the refusal boundary

| Option | Spend behavior | Offboarding boundary | Main trade-off |
| --- | --- | --- | --- |
| Atomic lifecycle check and bounded reservation | Refuses work when a bound will not fit | Closure serializes against admission | Requires shared durable coordination and conservative bounds |
| Usage reporting with alerts | Reports obligations after acceptance | Requires a separate closure mechanism | Preserves acceptance until closure, but does not enforce a ceiling |
| Delete user data first | Does not itself account for outstanding work | Can discard state needed by revocation and reconciliation | Early erasure complicates proving access closure |

Choose the first option for the stated workload. Its coordination burden belongs on the critical path because it decides whether the service is permitted to incur an obligation. Moving that decision to an asynchronous reporting pipeline changes the guarantee, even if the reporting is usually fast.

A partition forces the same decision into operational terms: if a worker cannot establish current lifecycle state and reserve capacity, refuse or defer new work. A locally cached allowance needs its own bounded reservation from the global allocation before it can safely authorize obligations; otherwise multiple workers can each spend the same apparent remainder. That added complexity needs a concrete throughput justification.

## Critical path and durable state

The following executable Python model shows the serialization boundary within one process. It deliberately holds credential metadata separately from payload deletion. It isn't a distributed or durable implementation: deployment requires transactions or equivalent conditional updates over the lifecycle record, credential state, and reservation ledger, plus durable dispatch and reconciliation. A process lock cannot provide those properties across workers or crashes.

```python
from dataclasses import dataclass, field
from threading import Lock


@dataclass
class TenantLedger:
    limit: int
    settled: int = 0
    closed: bool = False
    revoked: bool = False
    reservations: dict[str, int] = field(default_factory=dict)
    lock: Lock = field(default_factory=Lock, repr=False)

    def admit(self, operation_id: str, upper_bound: int) -> bool:
        if upper_bound <= 0:
            raise ValueError("upper_bound must be positive")
        with self.lock:
            if self.closed or self.revoked:
                return False
            if operation_id in self.reservations:
                if self.reservations[operation_id] != upper_bound:
                    raise ValueError("reservation parameters changed")
                return True
            committed = self.settled + sum(self.reservations.values())
            if committed + upper_bound > self.limit:
                return False
            self.reservations[operation_id] = upper_bound
            return True

    def begin_offboarding(self) -> dict[str, int]:
        with self.lock:
            self.closed = True
            self.revoked = True
            return dict(self.reservations)


ledger = TenantLedger(limit=1000, settled=730)
assert ledger.admit("existing-operation", 220)
assert not ledger.admit("too-large", 80)
outstanding = ledger.begin_offboarding()
assert outstanding == {"existing-operation": 220}
assert not ledger.admit("late-operation", 1)
```

A successful repeated admission here recognizes an existing reservation; it must never independently trigger another execution. The production dispatcher needs its own durable operation state and deduplication rule, including after settlement, when the operation no longer appears among outstanding reservations. Otherwise a retry can become a second job despite perfectly correct budget arithmetic.

The revocation flag models an authoritative authorization decision. Changing that flag doesn't reach into another process and retract its cached credentials, nor does it cancel an operation already sent downstream. Record separate acknowledgments where propagation is required, and keep cleanup pending until the defined access-closure conditions are met. If cleanup crashes halfway through, its persisted operation identifier and progress must let it resume without reopening admission or restoring tenant keys.

Test these interleavings before deployment: admission racing closure, two reservations competing for the last 50 units, a crash between reservation and dispatch, duplicate dispatch after a timeout, and cleanup interrupted after only part of the payload is erased. Each test should assert the invariant and the resulting durable state. Observability should distinguish admission refusal, revocation pending, unresolved reservation, and erasure pending; one generic offboarding status hides the very boundary the operator needs to inspect.

Deploy by routing every obligation-creating entry point through the gate before enabling ceiling enforcement. Inventory scheduled jobs and delayed retries alongside public requests. An overlooked worker is a bypass, even when the interactive path behaves correctly.

Audit the consumers.

## Rejected option and its valid use case

Reject asynchronous spend alerts as the enforcement mechanism for this workload. Alerts remain useful for a team that explicitly accepts variable consumption and prioritizes continued traffic, because they can support investigation and allocation review without putting coordination on every admission. That is a valid operating policy with a different promise. It still needs credential revocation before user-data deletion when offboarding closes access.

For the capped workload, completion requires evidence of closed admission, effective credential revocation, resolved obligations, and the recorded erasure outcome. The difficult decision is how much traffic to refuse while those conditions are established. Make that policy explicit before the first deletion runs.

## References

- OWASP Secrets Management Cheat Sheet. Security guidance on secret lifecycle management, revocation, authorization, and expiration; the budget ledger and offboarding transaction design above are proposed application policies. https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
