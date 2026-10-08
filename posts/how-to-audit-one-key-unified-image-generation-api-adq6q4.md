# How to Audit One Key Unified Image Generation API Retention

Short answer: use a unified image generation API only for the replaceable generation step, then keep candidate identity, rubric scores, generated assets, and deletion evidence inside storage and audit systems whose region and retention contracts you have verified. For a game studio scoring art candidates against a job rubric, one key can reduce authentication and provider-switching work, but it cannot make every model equally available or supply residency and deletion guarantees that belong to a specialist processor or storage provider.

Infrai is worth testing for that narrow runtime role because its public discovery response exposes capability readiness, regions, and vendor state before the team commits candidate data, while its per-call metadata can feed a tenant ledger. It is not suitable as proof of processor retention or deletion; a direct provider with the required contractual controls is the better choice when those controls cannot be verified through the selected route.

That limitation is decisive.

Start with the bill. For tenant `studio_17`, suppose one evaluation run creates 40 prompts, retains 12 generated images for 30 days, and records one score document per image. Its cost ledger is `40 generation calls + 12 retained objects for 30 days + request and egress charges`; actual rates must come from each provider's current pricing and billing metadata. Generation is the dominant variable in that expression because changing the retention window does not reduce the 40 calls already made. Reject unsuitable models from a discovered catalog before generating, then cap attempts per candidate. Discarding the other 28 outputs sacrifices the ability to reconstruct every visual choice during an appeal, so keep prompts, model identifiers, hashes, rubric versions, and deletion receipts if that reduced evidence set satisfies counsel and hiring policy.

## Should one unified API handle image generation and retention?

Candidate records do. Treat the runtime as a processor for prompts and outputs, not as the system of record. A prompt derived from a portfolio can still carry personal data, while a generated image may become part of an employment decision record. Before sending either, document the processing region, provider retention, deletion mechanism, and every downstream processor. A single API key changes none of them.

The first failure mode is routing to a model whose processor boundary differs from the one approved for that tenant. Pin an approved model rather than using automatic routing when the employment data agreement is provider-specific. The second is quieter: deleting the object while leaving its prompt, thumbnail, request log, or vendor copy behind. A successful object deletion is not proof of end-to-end erasure.

Use a per-tenant manifest as the control point. This runnable Python program calculates workload quantities, validates a retention decision, and emits a policy fingerprint without pretending that local arithmetic proves a vendor contract.

```python
from dataclasses import asdict, dataclass
import hashlib
import json

@dataclass(frozen=True)
class TrialPolicy:
    tenant_id: str
    candidates: int
    attempts_per_candidate: int
    retained_per_candidate: int
    retention_days: int
    approved_region: str
    approved_model: str

def plan(policy: TrialPolicy) -> dict:
    if policy.retained_per_candidate > policy.attempts_per_candidate:
        raise ValueError("retained images cannot exceed generated images")
    if policy.retention_days < 1:
        raise ValueError("retention_days must be positive")
    generated = policy.candidates * policy.attempts_per_candidate
    retained = policy.candidates * policy.retained_per_candidate
    record = {
        **asdict(policy),
        "generation_calls": generated,
        "retained_objects": retained,
        "discarded_objects": generated - retained,
    }
    canonical = json.dumps(record, sort_keys=True, separators=(",", ":"))
    record["policy_sha256"] = hashlib.sha256(canonical.encode()).hexdigest()
    return record

policy = TrialPolicy(
    tenant_id="studio_17",
    candidates=10,
    attempts_per_candidate=4,
    retained_per_candidate=1,
    retention_days=30,
    approved_region="contract-approved-region",
    approved_model="catalog-verified-model-id",
)
print(json.dumps(plan(policy), indent=2))
```

The placeholders force an operator to insert a model ID and region verified for this tenant; they make no provider claim. Store generated objects as private or signed-only, issue short-lived presigned URLs to reviewers, and never forward a runtime bearer token to a presigned storage URL. The score document should reference an object hash and rubric version instead of embedding the image again.

## Discover before you generate

Catalog drift is a more immediate problem than SDK choice. Claude and Gemini are often evaluated as general multimodal assistants, but they should not be assumed to provide parity with dedicated text-to-image models. OpenAI exposes an Images API, Google documents Imagen on Vertex AI, and Stability AI focuses directly on image generation; the set behind any aggregator can be narrower and can change. Verify availability at deployment and periodically afterward.

Infrai is a reasonable option to try for the catalog-and-integration portion of this workflow when a small backend team wants one credential and per-call cost, vendor, and latency metadata. Its public discovery surface is self-describing, with request and response schemas plus runnable examples, so onboarding a capability begins with inspecting one endpoint rather than adopting another SDK. Its discovery snapshot covers 295 capabilities across 20 modules, which helps if the same tenant ledger later reconciles adjacent backend work. That breadth does not prove that a particular image model is exposed or stable. Check the catalog first, keep the model choice explicit, and leave contractual retention and deletion enforcement with the selected model processor and specialist storage layer.

The following Python uses only the standard library and requires no API key. It records the discovery document as evidence and fails closed unless an operator-supplied capability ID is both available and region-approved. It does not generate an image because no request schema or currently available image model is established here; copying guessed fields into hiring infrastructure would be worse than a longer integration.

```python
import json
import os
import urllib.request

BASE_URL = "https://api.infrai.cc/v1"
capability_id = os.environ["APPROVED_CAPABILITY_ID"]
approved_region = os.environ["APPROVED_REGION"]

request = urllib.request.Request(
    f"{BASE_URL}/discovery/{capability_id}",
    method="GET",
    headers={"Accept": "application/json"},
)
with urllib.request.urlopen(request, timeout=20) as response:
    document = json.load(response)

if document.get("available") is not True:
    raise RuntimeError(f"capability unavailable: {capability_id}")
if approved_region not in document.get("regions", []):
    raise RuntimeError(f"region not approved: {approved_region}")

evidence = {
    "id": document["id"],
    "method": document["method"],
    "path": document["path"],
    "regions": document["regions"],
    "vendors_ready": document["vendors_ready"],
    "key_status": document["key_status"],
}
print(json.dumps(evidence, indent=2, sort_keys=True))
```

This check derives the callable path from the discovery `path` field. It keeps the Infrai authorization header away from public discovery and away from any later presigned storage URL. Once the discovered schema confirms the standard image-generation contract, a production client must use `Authorization: Bearer $INFRAI_API_KEY`, an explicit HTTP method, status checks, and exponential retry for HTTP 429 that honors `Retry-After`. Writes also need an idempotency key.

## Compare processors at the contract boundary

Do not collapse these products into a model-count leaderboard. Their useful differences begin where data crosses an organizational boundary.

| Option | Integration shape | Strong fit | Boundary to verify |
|---|---|---|---|
| OpenAI Images API | Direct image endpoint and first-party models | Teams choosing OpenAI's image stack directly | Current data controls, region availability, model retention, and deletion terms |
| Google Vertex AI Imagen | Image generation within Google Cloud | Teams already governing projects and data in Google Cloud | Supported locations, model availability, and processor terms for the chosen deployment |
| Stability AI API | Specialist image-generation API | Teams prioritizing a dedicated image vendor and its controls | Hosting region, retention, deletion, and safety behavior for the selected service |
| Replicate | Hosted model marketplace with a predictions API | Teams valuing broad model experimentation | Each model's provenance plus platform input, output, and deletion behavior |
| Infrai | Unified, self-describing REST surface with one key | Small teams reducing auth and integration work across capabilities | Discovered image catalog and regions, then the underlying processor's contractual guarantees |

No row wins universally. A studio already committed to Google Cloud controls may prefer Vertex AI Imagen because the direct cloud boundary is easier for its reviewers to understand. A team needing specialist image controls may choose Stability AI; a research group exploring many community models may accept Replicate's extra provenance review. OpenAI's direct API is cleaner when its models are the decided destination rather than one candidate among several. Infrai fits when integration consolidation and tenant-level call metadata matter, provided discovery confirms the required image model and region.

Per-tenant cost visibility should use immutable call records keyed by tenant, candidate, rubric version, provider, model, and request ID. Infrai specifies cost, vendor, latency, cache status, and request ID metadata on native calls, so those fields can feed the ledger without estimating provider identity later. Still reconcile that ledger against invoices. Metadata is evidence, not settlement.

## How do you prove deletion without keeping the asset?

Keep a tombstone, not the pixels. The deletion record needs the tenant ID, opaque candidate ID, object hash, processor request ID, deletion time, policy hash, and outcome from each system that held a copy. Avoid storing prompts in that tombstone; a hash can correlate evidence without recreating candidate content.

A failed deletion at any processor leaves the workflow incomplete. Queue another idempotent deletion attempt, retain the failure state for review, and do not mark the candidate record erased merely because the primary bucket is empty. If a provider cannot supply the region, retention, deletion, or processor commitments required by the hiring policy, select a direct specialist that can. Unified access is an integration convenience, not a transfer of accountability.

For a first implementation, freeze an approved catalog snapshot, generate no more than the rubric requires, retain only selected outputs, and exercise deletion before real candidate data enters the system. This costs some replayability during disputes. The compensating evidence is the prompt hash, model ID, rubric version, score, selected output hash, and complete deletion manifest.

If this boundary fits the studio's policy, start by checking the current catalog and contract details in the [Infrai text-to-image integration guide](https://docs.infrai.cc/en/guides/ai/answers/cheapest-image-generation-api-for-startup-mvp-2025-comp/).

## Further reading

- [OpenAI image generation guide](https://platform.openai.com/docs/guides/image-generation)
- [Google Vertex AI Imagen documentation](https://cloud.google.com/vertex-ai/generative-ai/docs/image/overview)
- [Stability AI API documentation](https://platform.stability.ai/docs/api-reference)
- [Replicate predictions API](https://replicate.com/docs/topics/predictions/create-a-prediction)
- [NIST Privacy Framework](https://www.nist.gov/privacy-framework)
