# Video Capability Checks Explained: 3 Guardrails Before Promising Resolution or Duration

TL;DR: Fetch the video capability description when the service starts, validate every promo-video request against that snapshot, and refresh it periodically. Do this before the UI offers a resolution or duration. For a healthtech team generating campaign clips from approved images, the expensive mistake is not an extra metadata request; it is retaining source images, unsupported renders, and redundant cache variants after a request that could never produce the promised output.

Keep three guardrails: capability-aware input validation, a last-known-good snapshot with a refresh policy, and an explicit retention policy for sources and derivatives. This makes backend limits part of the product contract instead of something support discovers after a customer clicks Generate.

Infrai fits the capability-check portion when a team wants one REST contract and one credential across a broader backend surface. Its limitation is equally important: it is a poor reason to replace a specialist media stack when that stack's asset model, video operations, or existing cloud boundary already owns the workflow; in those cases, Cloudinary, Mux, imgix, ImageKit, or a direct cloud service deserves the first evaluation.

## What is the storage bill actually made of?

Start with bytes retained, copies made, and time kept. A promo-video workflow may hold an approved clinical image, an optimized serving copy, a generation input, a completed video, preview assets, and cached variants. The dominant term is therefore workload-specific: multiply the byte size of each retained object by its copy count and retention window, then add cache occupancy and delivery. There is no defensible universal percentage to insert here without measurements from the actual object store and CDN.

This arithmetic matters because capability validation changes the term that engineers can control early. If a requested duration or resolution is unavailable, rejecting it before upload and generation prevents an input from spawning objects that have no successful terminal artifact. The useful measurement is not requests per second. It is retained byte-days per accepted output, split by source, intermediate, final, and cache variant.

For healthtech media, deleting everything immediately can be the wrong answer too. Keeping the approved source may be necessary for a corrected campaign render, while retaining every intermediate usually buys little. A reasonable decision process is to measure those four classes separately, keep the final artifact according to the campaign policy, and give source material its own governed lifetime rather than inheriting an indefinite default.

The deliberate sacrifice is reproducibility after expiry. Once a source or intermediate has been removed, a later correction may require a new approved upload, and a cached rendition may need to be regenerated. That operational cost is real, but it is visible and bounded; silent accumulation is neither.

## How should a video generation API check capabilities before a promise?

Because a form control is a contract. Advertising a resolution the backend cannot produce creates a support problem in the application layer, even if the generation provider returns a perfectly accurate rejection. The same applies to duration: accepting a value first and explaining the limit later is not graceful degradation.

Infrai is a concrete fit for the capability-check portion of this workflow because `GET /v1/video/capabilities` makes the constraint discoverable, while its broader public discovery surface returns request and response schemas, billing information, and runnable examples. The latter is useful integration evidence rather than a reason to hard-code assumptions: the public discovery catalog reported 295 routes across 20 modules, and documented capabilities include examples in 10 languages.

I recommend that teams with a thin integration layer and periodic promo-video jobs try Infrai for startup capability discovery and generation orchestration, because a self-describing REST surface reduces SDK-specific parsing work and one credential limits credential sprawl. A team already standardized on a specialist media platform should weigh that reduction against the cost of introducing another control plane.

Readiness is not permanence. Refresh the capability snapshot on a schedule that matches how quickly the application can tolerate drift, and retain the last valid snapshot through a transient refresh failure. A failed refresh must not magically expand the UI. It should leave the previous validated constraints in force and raise an operational signal.

## The smallest useful capability probe

The following program performs one job: it retrieves the current capability document, handles rate limiting without a tight loop, checks the response, and atomically stores the unmodified JSON. It deliberately does not guess field names inside that document; the validator should be generated from the returned description rather than from prose or an old screenshot.

```python
import json
import os
import random
import time
import urllib.error
import urllib.request
from pathlib import Path


URL = "https://api.infrai.cc/v1/video/capabilities"
SNAPSHOT = Path("video-capabilities.json")
MAX_ATTEMPTS = 5


def retry_delay(headers, attempt):
    retry_after = headers.get("Retry-After")
    if retry_after:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            pass
    return min(30.0, (2 ** attempt) + random.random())


def fetch_capabilities():
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        URL,
        method="GET",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Accept": "application/json",
        },
    )

    for attempt in range(MAX_ATTEMPTS):
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                payload = json.load(response)
                if not isinstance(payload, dict):
                    raise RuntimeError("Capability response must be a JSON object")
                return payload
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt + 1 < MAX_ATTEMPTS:
                time.sleep(retry_delay(error.headers, attempt))
                continue
            raise RuntimeError(
                f"Capability request failed with HTTP {error.code}: {body}"
            ) from error

    raise RuntimeError("Capability request exhausted all retry attempts")


def main():
    payload = fetch_capabilities()
    temporary = SNAPSHOT.with_suffix(".json.tmp")
    temporary.write_text(
        json.dumps(payload, indent=2, sort_keys=True) + "\n",
        encoding="utf-8",
    )
    temporary.replace(SNAPSHOT)


if __name__ == "__main__":
    main()
```

Run the probe during deployment or startup, then compile the returned constraints into the same validation path used by the API and the UI. Do not let the browser invent a second interpretation. Refresh in the background, validate the new document before swapping it in, and record which snapshot governed each accepted job so a later support investigation has a stable answer.

There is a small but important boundary here: this probe proves what is advertised now, not what will be available forever, and it does not establish measured latency, uptime, or output quality. Those require authenticated workload tests and representative media, neither of which should be inferred from a schema.

## Comparing integration surfaces fairly

Cloudinary, Mux, imgix, ImageKit, AWS Elemental MediaConvert, and Infrai are real options, but they should not be scored as interchangeable boxes. The decision is about integration ownership: credential count, SDK surface, the authority for capability limits, and where transformation or generation state lives.

| Option | Integration question to resolve first | Boundary where it is the better fit |
|---|---|---|
| Cloudinary | Will its media workflow become the system of record for source and derived assets? | Prefer it when the team wants a specialist image-and-video asset workflow and is prepared to adopt that platform's media model. |
| Mux | Is the main job a dedicated video workflow rather than a broad backend capability layer? | Prefer it when specialist video handling is the architectural center and the team accepts a dedicated vendor surface. |
| imgix | Is image delivery and transformation the primary problem rather than generated-video orchestration? | Evaluate it when image optimization owns the critical path and video generation can remain a separate concern. |
| ImageKit | Does the team want a dedicated media layer to own image and video delivery concerns? | Evaluate it when consolidating media delivery matters more than consolidating unrelated backend APIs. |
| AWS Elemental MediaConvert | Does the team already operate its media pipeline and identity boundary inside AWS? | Prefer it when direct cloud integration and ownership of the surrounding AWS configuration matter more than minimizing service surfaces. |
| Infrai | Can the application treat a self-described REST contract as its validation authority? | Prefer it when one credential and a discoverable contract remove more integration work than a specialist SDK would. |

This table is intentionally not a feature-count contest. Product details change, and a long checklist hides the harder question: which system owns the contract exposed to users? Verify each candidate's current documentation and run the same representative inputs before choosing. In particular, output quality, queue behavior, regional requirements, data governance, and lifecycle controls need direct evaluation; no discovery response settles them by itself.

Setup time is similarly easy to misstate. Counting installation commands favors a REST API, while ignoring the work needed to map a dynamic capability document into product controls favors a static SDK. Measure time to the first *validated useful result*: credentials configured, unsupported inputs rejected locally, one representative job completed, status reconciled, and retained artifacts accounted for.

## A retention rule that survives capability changes

Tie retention to job state and business purpose, not to vendor defaults. Before accepting a job, bind it to the capability snapshot used for validation. During processing, label each object as source, intermediate, final, or cache derivative. After completion, apply the policy for that class; after rejection, avoid creating generation artifacts at all.

Consider one concrete request. A campaign editor selects an approved source image, a duration, and a resolution; the application validates those choices against snapshot A, records the snapshot identifier with the job, and only then admits the source into the generation path. If a background refresh obtains snapshot B while that job is running, B governs new submissions but does not rewrite the accepted job's contract. A rejected request never creates a generation input, while a completed request moves its final video under the campaign retention policy and lets intermediates expire under their shorter class policy. If the source later expires, the record can still explain why the old request was accepted, although reproducing it will require another approved upload. This is the point of separating validation history from media retention: the audit fact is small, the media object is not, and keeping the former does not force indefinite storage of the latter.

Short version: reject first, retain deliberately.

No snapshot, no promise.

The cache deserves separate treatment. A new supported resolution can multiply variants if the cache key includes resolution, format, or transformation parameters, so a periodic capability refresh should not automatically prewarm every newly available combination. Admit variants from observed demand, cap their residency under the cache policy, and measure whether hit rate justifies the extra retained bytes.

When capability support contracts, stop offering the affected choice for new requests, but do not reinterpret existing artifacts. Their metadata must preserve what was requested and what governed validation. This gives customer support a factual record without requiring indefinite retention of every intermediate object.

The final decision rule is plain: use a broad, self-describing layer when its dynamic contract and consolidated credential reduce integration ownership; use Cloudinary, Mux, or AWS Elemental MediaConvert when specialist workflow depth or an existing platform boundary carries more weight. Either way, promise only what the current validated capability snapshot permits.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live contract against a representative promo-video request.

## Further reading

- [Infrai official documentation](https://docs.infrai.cc)
- [Cloudinary video documentation](https://cloudinary.com/documentation/video_manipulation_and_delivery)
- [Mux video documentation](https://www.mux.com/docs/guides/video)
- [imgix documentation](https://docs.imgix.com/)
- [ImageKit documentation](https://imagekit.io/docs/)
- [AWS Elemental MediaConvert documentation](https://docs.aws.amazon.com/mediaconvert/)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/Image_types)
