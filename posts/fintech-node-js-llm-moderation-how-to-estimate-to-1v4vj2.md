# Fintech Node.js LLM Moderation: How to Estimate Token Cost Before Classification

TL;DR: Put a deterministic admission record in front of the classifier, scoped to the tenant and pinned to a model-policy version. Estimate text and image charges from configuration rather than hard-coded vendor assumptions, reserve that amount against the tenant's moderation budget, require a JSON Schema-shaped finding, and then reconcile the reservation with authoritative usage returned by the runtime. The estimate decides whether work may start; it must never masquerade as the bill.

For a fintech code-review service, the useful unit is not “one moderation call.” It is a review attempt for a particular tenant, commit, policy version, and set of immutable inputs. That distinction gives finance a cost trail while keeping a malformed screenshot, retry, or oversized patch from quietly changing the denominator. **Admission and reconciliation are separate records.**

## How should Node.js estimate LLM moderation token cost before classifying images?

Text can be counted only with the tokenizer and message construction used by the selected runtime. Images follow a separate accounting rule that may depend on dimensions, detail settings, preprocessing, or another provider-specific unit. Structured output adds an upper bound for generated content, but a schema is a validation contract, not a promise that generation will consume the maximum.

One count cannot do both jobs.

So the preflight should expose its assumptions. For every tenant, store a policy version, currency, input and output rates, image estimator identifier, output cap, and safety margin. Rates belong in configuration with an effective date; embedding them in application code guarantees that historical estimates become hard to explain after a change.

There is another boundary worth drawing. Moderation determines whether submitted content violates a policy. The later code-review stage determines whether a change contains security, correctness, or compliance findings. Mixing those outputs into one schema makes “blocked content” indistinguishable from “clean code,” which is exactly the ambiguity an audit trail should remove.

## Build the admission envelope first

The Node.js edge can create the request and call a small internal estimator, but the logic below is Python because the ledger contract should be portable across workers. It uses caller-supplied counts and rate configuration; no universal text-to-token ratio or image formula is invented here.

```python
from dataclasses import dataclass
from decimal import Decimal, ROUND_CEILING
from typing import Iterable


@dataclass(frozen=True)
class RateCard:
    version: str
    currency: str
    input_per_million: Decimal
    output_per_million: Decimal
    image_unit_price: Decimal


def to_micros(amount: Decimal) -> int:
    return int((amount * Decimal(1_000_000)).quantize(
        Decimal("1"), rounding=ROUND_CEILING
    ))


def estimate_micros(
    *,
    input_tokens: int,
    max_output_tokens: int,
    image_units: Iterable[int],
    card: RateCard,
    margin_basis_points: int = 1_000,
) -> int:
    values = [input_tokens, max_output_tokens, margin_basis_points, *image_units]
    if any(value < 0 for value in values):
        raise ValueError("counts and margins must be non-negative")

    text = (
        Decimal(input_tokens) * card.input_per_million
        + Decimal(max_output_tokens) * card.output_per_million
    ) / Decimal(1_000_000)
    images = Decimal(sum(image_units)) * card.image_unit_price
    subtotal = text + images
    reserved = subtotal * (
        Decimal(10_000 + margin_basis_points) / Decimal(10_000)
    )
    return to_micros(reserved)


if __name__ == "__main__":
    example = RateCard(
        version="policy-2026-10-01",
        currency="USD",
        input_per_million=Decimal("2.00"),
        output_per_million=Decimal("8.00"),
        image_unit_price=Decimal("0.0015"),
    )
    print(estimate_micros(
        input_tokens=2_400,
        max_output_tokens=300,
        image_units=[4, 2],
        card=example,
    ))
```

Those sample rates are test fixtures, not a market price. In production, `input_tokens` must come from the tokenizer pinned to the selected model, and `image_units` must come from a named estimator that implements the selected runtime's published rule. Record both estimator identifiers beside the result. Otherwise the number is irreproducible.

The envelope should also carry `tenant_id`, `review_id`, a content digest, byte counts, media dimensions, and the requested output cap. Hashing identifies the exact input without making the ledger a second store for sensitive source code. Keep the source objects under their existing retention and access policy; a cost ledger does not need their contents.

## Make the decision atomic per tenant

Cost visibility fails when workers read a remaining budget, race, and each admit work against the same balance. A reservation must be an atomic conditional write. The exact database is less important than the invariant: within one transaction, insert one idempotent reservation only if the tenant-period total plus the new estimate remains within its configured limit.

Retries spend attention.

Use a uniqueness key such as `(tenant_id, review_id, attempt_kind)`. Consider a review containing a patch and two screenshots: the first delivery reserves against the exact content digest, the worker submits it, and the connection closes before usage reaches the caller. A queue retry with the same digest must attach to that attempt, because creating another reservation would exaggerate committed budget while releasing the first would understate work that may have run. If the user replaces one screenshot, the digest changes; if compliance publishes a new moderation policy, the policy version changes. Either change creates a distinct attempt even when the commit identifier stays fixed. This is why `review_id` alone is a poor idempotency key and why a transport request identifier is too narrow. The ledger needs enough identity to distinguish replay from new work without retaining the source or image itself.

The admission states can stay small:

| State | Meaning | Permitted next state |
|---|---|---|
| `estimated` | Inputs and policy were sized | `reserved`, `rejected` |
| `reserved` | Tenant budget was atomically held | `submitted`, `released` |
| `submitted` | One classified attempt began | `reconciled`, `unknown` |
| `reconciled` | Authoritative usage replaced the estimate | none |
| `unknown` | Outcome or usage could not be proved | `reconciled`, `released` after policy review |

Do not immediately release an ambiguous submitted attempt. A timeout says that the caller lacks an answer; it does not establish that no computation occurred. Keep the reservation in `unknown`, query whatever authoritative usage source is available, and apply a bounded retention policy. This is conservative accounting, and it prevents retries from making per-tenant costs look lower than the work actually requested.

The trade-off is explicit. A generous margin rejects more work near a budget ceiling but reduces under-reservation. A small margin improves admission but creates larger reconciliation deltas. Measure those deltas by policy version, then tune the margin from observed distributions rather than intuition.

## Require a narrow finding contract

JSON Schema should describe the moderation result independently of any runtime's request syntax. Keep the set of outcomes closed, reject undeclared fields, and cap strings and arrays so downstream review systems cannot receive an unbounded “structured” object.

```python
import json


FINDING_SCHEMA = {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "additionalProperties": False,
    "required": ["decision", "categories", "explanation"],
    "properties": {
        "decision": {"enum": ["allow", "block", "manual_review"]},
        "categories": {
            "type": "array",
            "maxItems": 8,
            "uniqueItems": True,
            "items": {"type": "string", "minLength": 1, "maxLength": 64},
        },
        "explanation": {"type": "string", "minLength": 1, "maxLength": 500},
    },
}


if __name__ == "__main__":
    print(json.dumps(FINDING_SCHEMA, indent=2, sort_keys=True))
```

Validate locally after generation even when a runtime advertises constrained output. A valid object may still be semantically uncertain, so `manual_review` is a first-class decision rather than an exception hidden in logs. Store the schema version and the normalized result with the attempt; never silently coerce an unknown category into a familiar one.

Four failure modes deserve separate counters: estimator failure, budget rejection, transport ambiguity, and schema rejection. Combining them into “classification failed” destroys the operational signal. Alert on ratios by tenant tier and policy version, while keeping tenant identifiers out of broad log streams unless access controls permit them.

## Compare mechanisms after defining the invariants

The architecture choice is narrower once the accounting rules are fixed.

| Mechanism | Cost visibility | Consistency boundary | Main failure mode |
|---|---|---|---|
| Synchronous preflight in the API | Immediate estimate per request | API process plus ledger transaction | Request latency and admission availability are coupled |
| Queue admission with reservation | Clear estimate per accepted job | Ledger commit before enqueue | Reservation can outlive a failed enqueue |
| Post hoc usage aggregation | Authoritative totals when usage arrives | External usage feed and local join keys | Cannot enforce a budget before work starts |

For fintech review traffic, queue admission is often the defensible baseline when reviews need not complete inline, but it requires an outbox or equivalent transactional handoff so “reserved but never enqueued” is recoverable. Synchronous admission is reasonable for low-latency paths if the ledger can meet the same availability target. Post hoc aggregation remains useful for reconciliation; by itself, it is too late to be an admission control.

No mechanism makes an estimate exact. The decision rule should therefore be operational: reserve before dispatch, reconcile from authoritative usage, and report both estimated and actual micro-units per tenant and policy version. Preserve the delta. It is evidence about estimator quality, not accounting noise to overwrite.

Keep the mismatch visible.

## Roll out with shadow reservations

Start by computing envelopes without rejecting requests. For one full tenant billing interval, compare estimates with authoritative usage, inspect the high-delta cases, and verify that retries collapse under the idempotency key. Then enable reservations for a small tenant cohort while leaving a manual override that is itself written to the ledger.

Next, inject failures between reservation and enqueue, after submission but before response, and during schema validation. The required outcome is boring: no duplicate spend record, no cross-tenant balance read, and every ambiguous attempt remains traceable. Only then should budget rejection become automatic.

**A cheap classifier with an untraceable denominator is not cheap accounting.** The durable design is the one that can answer which tenant requested the work, which assumptions admitted it, what actually ran, and why the two numbers differ.

## Sources

- https://json-schema.org/draft/2020-12/json-schema-core
- https://json-schema.org/draft/2020-12/json-schema-validation
- https://docs.cohere.com/docs/rerank-overview
- https://github.com/openai/whisper
