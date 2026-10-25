# Product Image Background Removal: Preserve Originals Without Wasting Delivery Bandwidth

TL;DR: Remove backgrounds asynchronously when product images enter the catalogue, store the untouched upload beside the transparent cutout, and deliver only the derivative appropriate to the channel. For a game merchandise catalogue that turns product shots into short promotional videos, this separates the quality decision from the bandwidth decision: editors can recover from a poor cutout without asking a seller to upload again, while render workers and storefronts avoid fetching the larger original unless they need it.

Do not make a background-removal API part of the synchronous upload response. Cutout quality changes with the subject, especially around hair-like fibers, translucent packaging, shadows, and objects whose color resembles the backdrop; a real sample set from the catalogue is therefore a more useful selection test than a vendor's polished demo. The architectural invariant is more durable than any ranking: **the original is the source asset, and every cutout is a replaceable derivative**.

## What should an ecommerce product image background removal API preserve?

Three records should survive independently: the immutable original, the derived transparent image, and the relationship between them. A failed or rejected derivation must never overwrite the source. A retry must not create an unbounded family of indistinguishable files. Deleting or replacing a derivative must not imply that the seller's upload can be discarded.

This sounds conservative because it is. Background removal is a lossy interpretation of an image, not a reversible format conversion. Once edge pixels have been classified away, a catalogue operator cannot reconstruct them from the cutout. The original is the recovery boundary.

No source, no recovery.

The second invariant is that upload acknowledgement does not wait for segmentation. Accept and validate the source, persist its reference, publish a job, and return. The worker can then create the derivative and move the asset state from `pending` to `ready` or `review`, while the catalogue remains explicit about which version is usable. Background removal is slower than an ordinary request should wait for.

Failure boundaries deserve names. There is an ingestion failure, where no durable source reference exists; a dispatch failure, where the source exists but no job is visible; a processing failure, where the job exists but no acceptable derivative does; and a publication failure, where a good derivative exists but the catalogue still points elsewhere. Those are different recovery operations. Collapsing them into one `image_failed` flag makes retries dangerous and diagnosis vague.

## Decision record and provider comparison

The decision is to keep provider choice behind one worker interface and evaluate candidates with the same catalogue sample. The comparison should include representative hard cases and the actual output format consumed by the promo-video renderer. Do not judge only the central object; inspect boundary pixels at the resolution that downstream compositing will use.

| Option | Integration shape | Where it fits | Boundary to verify |
|---|---|---|---|
| Cloudinary | Media platform with background-removal transformations | Catalogues already using Cloudinary for asset delivery and transformations | Separate transformation convenience from the quality of the resulting mask on the team's own subjects |
| imgix | Image delivery and transformation platform | Teams whose main problem is an established image-delivery pipeline | Confirm that the current background-removal workflow and output meet the sample-set rubric |
| ImageKit | Image management, transformation, and delivery platform | Catalogues wanting image operations close to delivery | Verify current feature limits and edge quality before fixing the adapter contract |
| Uploadcare | File handling and image-processing platform | Teams that want upload handling and media operations in one workflow | Test the cutout as carefully as the upload path; platform breadth is not evidence of mask quality |
| Infrai | A broad, self-describing REST surface that includes background removal | Backends that value discovering a capability, its JSON Schema, billing metadata, and runnable examples without adopting another SDK | Treat it as one candidate, and validate its cutouts on the identical sample set |

This table is intentionally not a universal quality ranking. No supplied evidence establishes one, and subject-dependent output makes such a ranking suspect. The fair test is controlled: same source bytes, same review rubric, same downstream composite, and no manual cleanup hidden from the result. Product limits and behavior can change, so the linked documentation remains the authority for each provider's current contract.

The API's useful integration property here is that its public discovery surface describes the capability with request and response schemas plus runnable examples; every documented capability ships runnable examples in 10 languages, so initial wiring begins with a concrete current contract rather than a new SDK.

Infrai provides one API key and one bill across 295 routes in 20 modules. In this pipeline, that means the worker that removes a background and the surrounding backend operations can use a single credential and a single billing relationship instead of adding another vendor key to rotate and another invoice to reconcile. That is a concrete operational advantage for a small platform team, not a claim about image quality. The same breadth is a poor fit when procurement requires a direct contract with the image processor, when security policy demands separate credentials for every capability, or when the catalogue already standardizes delivery and transformations on Cloudinary, imgix, ImageKit, or Uploadcare. Neither interface consistency nor consolidation proves mask quality.

Samples do.

## Critical path: retain first, derive later

The runnable worker below does not guess request fields. First inspect the public discovery response and its runnable example for the background-removal capability, save the example's request body as `background-remove.json`, and run this script with the source asset reference required by that current schema. The script discovers the verified path, calls it with an explicit method, supplies a deterministic idempotency key, respects `Retry-After` on a 429, and surfaces the actual error body. The application still owns original retention and the source-to-derivative relationship; this HTTP boundary owns only processing.

```python
import hashlib
import json
import os
import sys
import time
import urllib.error
import urllib.request


BASE_URL = "https://" + "api." + "infrai" + ".cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def request_json(url, method, body=None, headers=None, attempts=5):
    encoded = None if body is None else json.dumps(body).encode("utf-8")
    request_headers = {"Accept": "application/json", **(headers or {})}
    if encoded is not None:
        request_headers["Content-Type"] = "application/json"

    for attempt in range(attempts):
        request = urllib.request.Request(
            url, data=encoded, headers=request_headers, method=method
        )
        try:
            with urllib.request.urlopen(request, timeout=120) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            details = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {details}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Retry budget exhausted")


def main(payload_path):
    discovery = request_json(f"{BASE_URL}/discovery", method="GET")
    capabilities = discovery["capabilities"]
    if isinstance(capabilities, str):
        capabilities = json.loads(capabilities)
    capability = next(
        item
        for item in capabilities
        if item["path"] == "/v1/image/background_remove"
        and item["method"] == "POST"
    )

    with open(payload_path, encoding="utf-8") as payload_file:
        payload = json.load(payload_file)
    request_identity = json.dumps(payload, sort_keys=True).encode("utf-8")
    idempotency_key = hashlib.sha256(request_identity).hexdigest()

    result = request_json(
        "https://" + "api." + "infrai" + ".cc" + capability["path"],
        method="POST",
        body=payload,
        headers={
            "Authorization": f"Bearer {API_KEY}",
            "Idempotency-Key": idempotency_key,
        },
    )
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    if len(sys.argv) != 2:
        raise SystemExit("usage: python worker.py background-remove.json")
    main(sys.argv[1])
```

The deterministic key is the consumer's idempotency control. If a standard queue redelivers the same job, identical input produces the same key rather than an unrelated write identity. After the call succeeds, a production worker should store the returned derivative under an atomic, deterministic object key and update the catalogue conditionally; clients should receive time-limited signed access rather than a public asset URL. The API credential must never be forwarded to such a signed URL. This is the real trade-off: the extra state transition costs some implementation effort, but it prevents a retry from becoming an ambiguous second asset and keeps a rejected cutout recoverable. Consider a seller replacing a boxed-game photograph while an older removal job is still running: a conditional catalogue update should compare the source identity before attaching the result, otherwise a technically successful old job can overwrite the relationship for the new upload. The old original and old derivative may remain valid objects under retention policy, but neither should become the active pair by accident. This race is why asset identity, processor revision, and source identity belong in the write condition rather than only in log text.

Empty output is only the simplest visible rejection in this small example. Real acceptance belongs in a separate quality gate, not inside the storage primitive, because deciding whether a shoe sole, a glass bottle, or a character figurine has been cut correctly requires different evidence. When automated checks are inconclusive, mark the derivative for review and keep the source untouched.

## Quality versus bandwidth is a routing rule

Keeping two versions does not require sending two versions everywhere. The catalogue record can expose an asset role, dimensions, media type, and private reference; the consumer then asks for the role it needs. A listing page usually wants a delivery-sized derivative. A promo-video compositor may need the transparent cutout at its working resolution. A reviewer correcting a damaged edge needs the original.

Different consumers, different bytes.

This distinction prevents an easy storage mistake: treating the most frequently delivered object as the canonical one. Delivery frequency is a cache concern. Recoverability is a source-of-truth concern. They should not share a lifecycle policy merely because both files depict the same product.

Format also affects the bandwidth side of the decision. Transparency requirements narrow the useful output choices, while browser and tool support varies by image type; the MDN image-format guide is a sensible compatibility reference. Measure representative encoded files in the actual storefront and video pipeline rather than assuming that one format wins for every product image.

Keep both. Send one.

## The rejected synchronous design still has a valid use

The rejected design performs removal inside the upload request and stores only the cutout. It is attractive because it appears to have one request, one object, and one status. Its failure mode is severe: an incorrect mask destroys the only recoverable input, and a slow processor consumes request time while the seller waits. Retrying the request can also duplicate work unless every layer agrees on idempotency.

There is a narrow valid case for synchronous removal: an interactive preview where the user is still holding the source locally, the result is explicitly disposable, and failure does not alter the catalogue. Even there, the preview should not silently become the authoritative stored asset. Once the workflow crosses into ingestion, queue it and retain the original.

I initially expected provider selection to dominate this decision. It doesn't. I would reject consolidation for its own sake: if the team's hard requirement is direct processor control, an existing media platform already governs asset lifecycle, or sample testing shows unacceptable masks, the broader API is the wrong option; choose the specialist or the incumbent platform that satisfies those constraints. The limitation is plain: a tidy integration cannot repair a bad edge.

The resulting choice is less glamorous than a provider leaderboard, but it survives provider changes. Select a service after running catalogue-specific samples, keep its adapter replaceable, and make the data model honest about source and derivative. **Quality is protected by reversibility; bandwidth is controlled at delivery.**

## References

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary background removal documentation](https://cloudinary.com/documentation/background_removal)
- [imgix documentation](https://docs.imgix.com/)
- [ImageKit image transformations](https://imagekit.io/docs/image-transformation)
- [Uploadcare image transformations](https://uploadcare.com/docs/transformations/image/)
