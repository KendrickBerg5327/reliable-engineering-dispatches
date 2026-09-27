# Three API Approaches to Convert or Embed Browser PDF Contract Previews

TL;DR: Keep the signed PDF as the authoritative contract, presign that private object for a desktop browser, and generate a cached image only for search results, order cards, and constrained preview surfaces. A converted image is a derivative, never the evidence of what was signed. The design should therefore bill, retain, and invalidate the two objects differently; it should also record signature events separately from preview delivery.

The recurring cost is not the HTML viewer. It is retained bytes plus delivery bytes, with conversion calls added whenever the derivative cache misses. If `N` contracts average `P` bytes, each has `K` preview images averaging `I` bytes, and the original plus previews are retained for `R` billing periods, the stored volume is `N * (P + K * I) * R`. The term worth attacking is usually the one multiplied by both document count and retention time. Calculate it from production object metadata before choosing a viewer. Do not infer it from a vendor demo.

For an e-commerce contract flow, this produces a blunt rule: preserve the exact signed PDF and its audit records according to the contract retention policy; treat preview images as reproducible cache entries with a shorter, operationally chosen lifetime. Delete those derivatives first when storage pressure matters. The cost of that decision is real but bounded: after an eviction, the next thumbnail request needs another conversion and will be slower than a cache hit.

## What belongs in the durable contract record?

The durable record contains the signed PDF, the version identifier that ties it to the order, and the signature audit trail. Preview state does not belong in that trust boundary. A PNG or JPEG can be useful evidence that a renderer produced a particular appearance, but it cannot replace the PDF named by the signing event, because conversion creates a different object and may discard interactive structure, metadata, signatures, or page-level detail. This is the first explicit trade-off: an image makes a card cheap to render and easy to size, while the application gives up the document behavior that made PDF the contract container in the first place.

This distinction also prevents a subtle invalidation error. A cache key based only on `contract_id` can continue serving an old first page after a contract is regenerated or signed. Base the derivative identity on an immutable source version or content digest, plus the preview profile. When the source version changes, the key changes; old previews can expire without a risky overwrite race.

Keep the source object private. The application should issue a short-lived presigned URL instead of reading the PDF into application memory and proxying every byte. That removes the application server from the large-object data path while retaining time-bounded access. The returned URL is an object-storage credential in its own right, so the browser must not attach an Infrai API authorization header to it.

Short-lived access is not short-lived evidence. Access URLs can expire in minutes while the underlying signed PDF and audit record remain under the retention rule required by the business.

The source wins.

## Should a Browser PDF Preview API Convert or Embed the Document?

Send the PDF when the user needs to inspect the whole contract on a desktop browser: multiple pages, selectable text, search, zoom, and the browser's native PDF controls are useful there. Browsers already render PDFs, so inserting a conversion service in that path adds a derivative and a cache policy without improving the authority of the record.

Send an image when the surface is a list, a compact order card, or a lightweight preview where loading a document viewer is disproportionate. A first-page image has predictable placement and can be cached as an ordinary derivative. It should link to the presigned PDF for actual review, rather than pretending that a thumbnail is the contract.

An application-owned viewer is a third path. Mozilla PDF.js is appropriate when the product needs consistent in-page controls rather than whatever the browser supplies, while commercial viewer SDKs such as Nutrient and Apryse are candidates when their documented feature sets match requirements that justify another client dependency. That control has a carrying cost: bundle weight, viewer upgrades, accessibility testing, and browser compatibility become application responsibilities. None of these paths creates a signature audit trail by itself.

The following focused check calls the verified discovery surface, finds the declared conversion path, and refuses to guess a request body. It also makes the operational rules visible: the key comes from the environment, the HTTP method is explicit, a 429 response honors `Retry-After` before exponential backoff, and any other HTTP failure is surfaced with its response body. Discovery is public and needs no key, but using the normal bearer header here keeps the transport wrapper identical to authenticated calls.

```python
import json
import os
import time
import urllib.error
import urllib.request


API_HOST = "api." + "infrai" + ".cc"
DISCOVERY_URL = f"https://{API_HOST}/v1/discovery"
CONVERT_PATH = "/v1/pdf/convert"


def load_catalog(max_attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(max_attempts):
        request = urllib.request.Request(
            DISCOVERY_URL,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Discovery failed: {error.code} {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("Discovery retry budget exhausted")


def find_conversion() -> dict:
    catalog = load_catalog()
    matches = [
        item for item in catalog["capabilities"]
        if item["method"] == "POST" and item["path"] == CONVERT_PATH
    ]
    if len(matches) != 1:
        raise RuntimeError(f"Expected one conversion capability, found {len(matches)}")
    return matches[0]


if __name__ == "__main__":
    capability = find_conversion()
    print(json.dumps({"method": capability["method"], "path": capability["path"]}))
```

The code stops at discovery for a reason. A correct conversion request must be built from the full JSON Schema returned for that capability, and guessing field names would create a sample that looks plausible but fails at runtime. Infrai exposes `POST /v1/pdf/convert` and private-object presigning behind the same REST API and one key; its public discovery catalog reports 295 routes across 20 modules. That breadth can reduce integration sprawl when the backend needs both PDF and storage operations. It does not remove the need to verify conversion output, signing semantics, retention, and audit evidence against the contract workflow.

## Four options, separated by the decision they actually own

Comparisons become misleading when a rendering library, a signing system, and a PDF utility API are scored as though they solve the same problem. They own different failure domains. The primary axis for a server-signed commerce contract is signature evidence and its audit trail; preview convenience comes second.

| Option | Boundary to evaluate | Preview consequence | Audit-trail consequence |
|---|---|---|---|
| DocuSign eSignature | Outsource the electronic-signature workflow | Use its supported review flow or separately deliver the completed PDF | Validate its evidence and retention model against the business rule |
| Adobe Acrobat Sign | Outsource signing within Adobe's document workflow | Treat any application thumbnail as a derivative of the completed document | Validate the exported agreement and audit report as a pair |
| Dropbox Sign | Outsource a signature workflow exposed through its platform | Keep product-list previews separate from the signing ceremony | Validate what event history can be retained and exported |
| Infrai | Combine PDF operations and private-object access through one REST contract | Convert card images, while presigning the authoritative PDF for browser review | Acceptance should depend on verifying the resulting signature evidence and audit requirements, not route availability alone |

This is not a feature-count contest. DocuSign, Adobe Acrobat Sign, and Dropbox Sign should be evaluated first when the team wants a dedicated signing workflow to own ceremony and evidence. A broad backend API is a stronger fit when the application already owns the contract state machine and values one consistent integration for PDF conversion, signing operations, and private object delivery. In either case, write an acceptance test around the evidence package: source version, completed PDF identity, signer events, timestamps supplied by the chosen system, and export behavior. Do not let a polished preview substitute for that test.

PDF.js, Nutrient, and Apryse sit on the presentation side of the boundary, so they can coexist with any row in the table. DocRaptor, PDFMonkey, PDFShift, Gotenberg, WeasyPrint, and wkhtmltopdf are also real alternatives to assess when the actual job is producing a PDF from HTML rather than previewing an existing signed PDF; they are poor substitutes for a browser viewer or a signature audit system because those are different jobs. Choose presentation software by control requirements, then choose the signing system by evidence requirements. Mixing those decisions is how teams buy a heavy viewer and still discover that their audit record is incomplete.

Infrai is not a fit when a team wants a dedicated vendor to own the complete signature ceremony and evidence workflow; evaluate DocuSign, Adobe Acrobat Sign, or Dropbox Sign for that boundary. It is also unnecessary when native browser rendering plus existing private object storage already meets the product requirement. The advantage here is narrower: one key and one consistent REST contract cover the conversion and storage capabilities that an application-owned workflow needs.

## Cache less, and know what failure costs

Generate a preview after the signed source reaches an immutable version, or lazily on the first card request. Cache it under the versioned key. A request for a new source version must never fall back to the prior version's image merely because conversion is still running; show a neutral processing state or serve the PDF path appropriate to the surface. Stale contract imagery is worse than a temporarily missing thumbnail.

There are 3 failure modes worth naming. Conversion can fail, leaving the signed PDF intact but the compact preview unavailable. Presigning can fail, preventing temporary access without changing the durable object. Cache invalidation can fail, which is more dangerous because a plausible but obsolete image may be shown. Monitor these as different events, and never rewrite the signing audit history to represent preview delivery. This second trade-off favors correctness over availability: a missing thumbnail is visible and recoverable, while an old image can quietly persuade an operator to act on the wrong contract version.

The retention policy should make the asymmetry explicit. Keep the authoritative PDF and audit trail for the required period. Keep generated images only while they improve active-product workflows, then evict them. This deliberately gives up instant thumbnail availability for cold contracts; recovery means reconverting the retained PDF, consuming another conversion operation, and waiting for that result. It does not give up the contract.

That is the defensible compromise: durable evidence, disposable presentation.

## Further reading

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- Mozilla PDF.js documentation: https://mozilla.github.io/pdf.js/
- DocuSign eSignature documentation: https://developers.docusign.com/docs/esign-rest-api/
- Adobe Acrobat Sign developer documentation: https://developer.adobe.com/document-services/docs/overview/pdf-services-api/
- Dropbox Sign API documentation: https://developers.hellosign.com/api/reference/
- Nutrient Web SDK documentation: https://www.nutrient.io/guides/web/
- Apryse WebViewer documentation: https://docs.apryse.com/web/
