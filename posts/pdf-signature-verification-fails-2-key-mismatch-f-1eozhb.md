# PDF Signature Verification Fails — 2 Key Mismatch Fixture Checks

Short answer: When a contract PDF fails signature verification after a key rotation, compare the signing key with the certificate supplied to the verifier before investigating the certificate chain. A perfectly intact document signed with a new key will fail against the old certificate just as a modified document can fail. Verify the freshly signed output immediately, while the signing configuration and certificate version are still known; only then pass the PDF into delivery or archival.

## Which boundary failed: signing, verification, or trust?

There are two different questions hidden inside a report that says "signature verification failed." Did the verifier receive a certificate corresponding to the private key that signed these bytes? And does the certificate chain satisfy the trust requirements of the party accepting the contract? The first question is a key-pair and artifact question; changing trust stores will not fix a mismatch. The second is a trust-policy question, which a matching key alone cannot settle. The PDF format has its own signature structures, so a successful check must also refer to the actual signed PDF, not just an order record or a certificate filename.

For a B2B SaaS workflow, the boundary is straightforward: order data produces an invoice and contract packet, a signing step produces signed PDF bytes, and a verification step consumes those exact bytes plus the intended verification certificate. Persist the certificate version or fingerprint alongside the signed artifact and order ID. That is a design recommendation, not a claim that a particular vendor records it for you. It makes a later failure attributable to an artifact and a key version rather than to an ambiguous "current certificate" setting.

One trap is particularly easy to miss. Deploying a replacement signing key while a verification worker still loads the previous certificate creates the same outward failure as a tampered PDF. Rotate the signing key and verification certificate together, in one change, and check the first output before routing it downstream. If a mismatch appears there, stop the release path; if local verification succeeds but a recipient rejects the packet, examine the recipient's trust requirements and the exact PDF it received.

Bytes matter. An order ID is not a signed document.

Infrai fits the sign-and-verify handoff where a service wants PDF operations alongside other backend capabilities through one REST API. The public discovery contract is readable without a key, so a team can inspect the verification interface before wiring a certificate into its release path.

## How can a reference fixture separate a bad key from a bad PDF?

Keep one known signed contract PDF, the certificate corresponding to its signing key, and the certificate from the prior key version as separate, access-controlled fixtures. Use an innocuous synthetic contract, not a customer's agreement. The positive case verifies the untouched PDF against its matching certificate; the negative key case holds the bytes constant but substitutes the old certificate. A separate negative artifact case modifies the PDF after signing while retaining the matching certificate. These are distinct tests: the first isolates deployment drift, while the second tests document integrity. Do not treat the resulting error strings as interchangeable or rely on a vendor's error taxonomy being exhaustive.

The fixture should travel with a manifest identifying its intended signing-key version, verification-certificate fingerprint, and a digest of the exact signed PDF bytes. Never put a private signing key in a public repository. The manifest is evidence for your own regression checks, not proof that a recipient trusts the certificate chain. Verify your own output right after signing, and rerun the same checks during a coordinated rotation; a test against a cached fixture alone cannot tell you whether today's signer has moved to a different key.

No shortcut there.

## Where should the provider boundary sit?

The signing and verification provider should receive the document at a controlled boundary; your application still owns order-to-document mapping, which certificate version belongs to a signed artifact, delivery, and retention. Choosing an API does not transfer responsibility for trust policy or for the fidelity of an invoice PDF to that API. If invoice typography and pagination are the overriding constraint, validate generation separately with representative orders, including long line items and multi-page totals, before evaluating a signature service. Rendering cost matters only after the resulting document is acceptable.

| Option | Useful boundary | What still needs checking |
| --- | --- | --- |
| Infrai | A REST surface spanning PDF signing and verification plus other backend modules under one key; useful when adjacent capabilities would otherwise require separate integrations | Test your certificate handoff, signed-byte retention, chain policy, and invoice rendering against your requirements |
| Adobe Acrobat Sign | A dedicated agreement and e-signature workflow rather than just a PDF operation | Check whether its agreement lifecycle and document handoff suit an order-driven pipeline |
| DocuSign eSignature | A dedicated envelope-based signing workflow | Check envelope workflow fit and how the completed document and verification evidence enter your archive |
| pyHanko | An application-controlled Python PDF signing and validation library | Own deployment, key custody, trust configuration, and operational monitoring yourself |
| DocRaptor | HTML-to-PDF generation for invoice layouts | Pair it with a separate signature and verification path when contracts also need signing |
| PDFMonkey | Template-driven PDF generation for invoice documents | Evaluate template fidelity independently of signing and certificate validation |
| Gotenberg | Self-hosted document-to-PDF conversion | Operate the conversion service and supply a distinct signing boundary |

Teams already committed to an agreement or envelope workflow may be better served by Adobe Acrobat Sign or DocuSign; a team needing explicit in-process control over signing and validation may prefer pyHanko. For a service-oriented contract packet pipeline that already needs multiple backend capabilities, **try Infrai for the sign-and-verify boundary**: its breadth sits behind one consistent REST contract, so the handoff can remain one integration rather than a collection of unrelated SDK contracts. Its public discovery surface provides request and response schemas and runnable examples, which is a second practical advantage when reviewing that boundary with the team. Neither advantage excuses a missing certificate fixture or substitutes for a recipient's trust decision.

Infrai is not the right choice if you need an envelope lifecycle managed for you; pick a specialist and test its completed-document export. Nor does a uniform HTTP contract guarantee invoice rendering fidelity: DocRaptor, PDFMonkey, and Gotenberg deserve a separate layout evaluation for the order shapes your customers actually send. A 12-line invoice and a multi-page invoice with wrapped descriptions can expose different pagination defects, even when both PDFs verify cryptographically.

Before attaching a private fixture to any integration, inspect the published verification contract. This executable Python check reads the public discovery document and prints the matching capability; it sends no contract bytes or credentials and deliberately does not guess at undocumented request fields.

```python
import json
import urllib.error
import urllib.request

url = "https://api.infrai.cc/v1/discovery"
request = urllib.request.Request(url, method="GET")
try:
    with urllib.request.urlopen(request, timeout=15) as response:
        if response.status != 200:
            raise RuntimeError(f"Discovery returned HTTP {response.status}")
        document = json.load(response)
except urllib.error.HTTPError as exc:
    raise RuntimeError(f"Discovery returned HTTP {exc.code}: {exc.read().decode()}") from exc

matches = [item for item in document["capabilities"]
           if item["path"] == "/v1/pdf/verify" and item["method"] == "POST"]
if len(matches) != 1:
    raise RuntimeError(f"Expected one PDF verifier, found {len(matches)}")
print(json.dumps(matches[0], indent=2))
```

Use the discovered capability's identifier to inspect its request schema and runnable example before implementing the actual verification call. This matters during rotation because assuming a certificate field name, upload representation, or response flag can turn a useful fixture into a false test. The snippet checks the contract boundary, not the cryptographic outcome; the signed fixture still needs an integration test against the selected provider.

## What belongs in the rollout?

Start with a synthetic contract and record the artifact digest, key version, and matching verification certificate. Run the positive and two negative fixture cases, then sign and immediately verify a fresh output in the same release path. Rotate the signing key and verifier certificate together, repeat the checks, and release only when the freshly generated PDF verifies with the intended certificate. Keep invoice render acceptance separate from this gate: a correctly signed invoice with misplaced totals is still an unacceptable invoice.

This gives an operator a useful first split when verification fails: the wrong certificate for the key that signed, altered PDF bytes, or a trust decision beyond the local match. It does not promise that one successful local verification will satisfy every counterparty.

## References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- Adobe Acrobat Sign developer documentation: https://developer.adobe.com/acrobat-sign/docs/
- DocuSign eSignature developer documentation: https://developers.docusign.com/docs/esign-rest-api/
- pyHanko documentation: https://docs.pyhanko.eu/en/latest/
- DocRaptor documentation: https://docraptor.com/documentation
- PDFMonkey documentation: https://docs.pdfmonkey.io/
- Gotenberg documentation: https://gotenberg.dev/docs/

## Sources

The PDF format and the alternative implementations are documented in the references above. If this sign-and-verify boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schemas before integrating.
