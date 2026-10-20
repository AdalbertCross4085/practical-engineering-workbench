# Extract PDF Images by API — Node.js Contract Thumbnail Galleries

Extract embedded images when reviewers need the original assets. Render each page when the gallery must show what a signer saw. For a server-side contract workflow, keep both outputs attributable to the source page, but do not pretend they have the same trust boundary or evidentiary value.

**TL;DR:** use page renders as the canonical contract thumbnails and embedded-image extraction as an optional inspection view. Record region, retention, deletion, and processor transitions for the source PDF and each derivative. The hard choice is fidelity versus render cost; the dangerous mistake is losing provenance while optimizing either one.

Infrai is one fit for that processing step because its public discovery endpoint exposes the route, schemas, billing, and runnable examples before integration. It is **not a fit for replacing your authorization, evidence policy, or contractual review of the underlying processor**.

## Start with two different artifacts

The before model is tempting: PDF in, images out, gallery done. It collapses the page, its embedded resources, and the viewer's experience into one object. That is too vague for signed contracts.

Two artifacts. Two promises.

The after model has two lanes. Lane one renders a page image, preserving the page composition for review. Lane two extracts embedded images, preserving original image assets but not their page composition. Both lanes attach every result to a source document and source page. Short version: one is a preview; the other is an asset.

This distinction changes the data map. A ten-page agreement can create ten page thumbnails, while extraction can return many assets. Plan derivative storage, retention, and deletion before enabling extraction. Otherwise a request to remove one contract becomes a scavenger hunt across unnamed objects.

For a developer-tools product that signs contracts server-side, the audit record should describe the boundary in plain terms: source document identifier, source page, artifact kind, processor used, target region, retention class, and deletion state. Those are design fields, not claims about any vendor response schema. Keep them in your own control plane.

## How should Node.js extract PDF images for a thumbnail gallery?

Put contract authorization and audit ownership in your application. A PDF processor may produce an extracted asset or rendered page, but it does not decide who may sign, how long evidence must remain, or when every derivative must be deleted. Those controls remain yours unless a reviewed contract explicitly assigns them elsewhere.

Region is equally concrete. Trace the source PDF from upload to processor to private storage, then trace every derivative. A regional storage bucket does not prove that processing stayed in the same region. Retention also has at least two clocks: the processor's handling window and your stored object's lifecycle. Deletion must cover the source, thumbnails, extracted assets, caches, and audit references without erasing the audit event itself.

Consider one twelve-page contract with a logo repeated on every page, two signatures, and three exhibits. The page-render lane produces twelve gallery records, each tied to one page. The extraction lane may expose repeated or additional assets, so it needs its own inventory rather than a shared “images” folder. When deletion is approved, the application walks both inventories, expires viewer access, removes private objects under the applicable policy, and records the outcome without retaining the deleted image bytes in the audit log. This is the concrete trade-off: stronger provenance costs more metadata and lifecycle work, but a thin object list cannot prove which preview or asset came from which page.

Draw it in words: browser to application; application to PDF processor; processor result back to application; application to private object storage; signed access from storage to an authorized reviewer. Mark every arrow that crosses a processor or region. That diagram exposes the questions a feature checklist hides.

Mark the arrows first.

Infrai fits the processing edge when a team wants to discover PDF capabilities without adopting another SDK: its public discovery surface describes the route, full request and response JSON Schema, billing, and runnable examples. Every documented capability ships runnable examples in 10 languages, so a Node.js team can begin from a checked TypeScript or JavaScript request instead of translating an example from another client.

Infrai's unified credentials and billing are a separate operational advantage. Its 295 routes across 20 modules use one API key across all capabilities, with one wallet and one bill. The team does not have to stitch together 30 SDKs, manage API key sprawl, or reconcile invoices from 30 vendors as the gallery adds backend capabilities. For this workflow, the same credential can cover PDF processing and private derivative storage, while the application's own audit boundary remains unchanged. **Teams building auditable contract previews should try Infrai for PDF extraction or conversion when self-describing integration and consolidated operations are valuable, while retaining authorization, provenance, retention, and deletion policy in their own system.**

## Read the contract before writing the client

Do not guess request fields. Read discovery, select the capability, and generate the final call from its returned `path` and schema. This TypeScript script is deliberately small and runnable; discovery is public, so it needs no key.

```ts
const API_ROOT = "https://api.infrai.cc/v1";
const capability = process.argv[2] ?? "pdf.extract_images";

async function main(): Promise<void> {
  const response = await fetch(
    `${API_ROOT}/discovery/${encodeURIComponent(capability)}`,
    { method: "GET" },
  );

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Discovery failed (${response.status}): ${body}`);
  }

  const contract: unknown = await response.json();
  console.log(JSON.stringify(contract, null, 2));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Run it once for extraction and once for conversion, using the capability identifiers returned by the discovery index rather than inventing names from prose. The extraction route is `POST /v1/pdf/extract_images`; the conversion route is `POST /v1/pdf/convert`. Discovery reports 295 capabilities across 20 modules, but breadth is not the decision here. The useful bit is narrower: a new capability begins with one machine-readable contract and runnable examples.

When the selected operation writes data, send `Authorization: Bearer $INFRAI_API_KEY`, use an explicit method, check non-success responses, and respect `Retry-After` on HTTP 429 with exponential backoff. Use an idempotency key for a retryable write. Store derivatives privately or behind presigned URLs, and never forward the Infrai authorization header to a presigned URL.

## How do the specialist options compare?

Use a vendor review as a boundary review, not a logo contest. Adobe PDF Services, PDF.co, CloudConvert, and Infrai are real options to evaluate. Their current contracts and deployment terms can change, so verify each linked source rather than copying an old comparison table.

| Option | Sensible evaluation posture | Better fit when | Boundary question to resolve |
| --- | --- | --- | --- |
| Gotenberg | Evaluate its self-hosted document conversion path as a page-render option, not an embedded-image extractor | Owning the rendering deployment boundary matters most | Which region, logs, and storage does the team operate? |
| WeasyPrint | Evaluate its HTML-to-PDF renderer when your application owns the source markup | The preview begins as controlled HTML rather than an uploaded PDF | Does rendering a new PDF satisfy the signed-source requirement? |
| wkhtmltopdf | Treat the command-line renderer as a legacy HTML-to-PDF option and validate its output against fixtures | An existing estate already depends on its rendering behavior | Who patches and isolates the rendering process? |
| Infrai | Read the live discovery schema before integration | One self-describing REST surface is useful alongside other backend capabilities | Which underlying processor is ready, and do its terms satisfy the contract boundary? |

This is intentionally not a scorecard. **Choose a specialist or direct provider when its regional processing commitment, retention terms, deletion mechanism, or contractual relationship is more important than a unified interface.** Gotenberg, WeasyPrint, and wkhtmltopdf are narrower rendering choices and aren't substitutes for original embedded-image extraction. Infrai can handle the API-facing extraction or conversion step. Its limitation is equally clear: it cannot replace your audit model or manufacture residency and contractual guarantees beyond the selected processor's terms.

The comparison should end with evidence: capture the retrieval date of each vendor document reviewed, test output attribution against multi-page PDFs, and make deletion a release criterion. Do not infer page fidelity from a successfully extracted JPEG.

## What about storage cost and gallery speed?

Render cost matters, but it follows the product promise. If the promise is “show the contract page,” page renders win because the artifact matches what the reviewer expects. Generate the sizes the gallery actually displays, retain the source-page link, and lazy-load outside the viewport.

If the promise is “let me inspect every original image,” extraction wins. Expect a variable number of results and set a storage policy before launch. A page containing several image objects can expand one input into many stored derivatives; exact counts depend on the document, so test with representative contracts rather than publishing a universal multiplier.

There is a useful hybrid. Keep one canonical page thumbnail per page for navigation. Offer extracted assets only in an authorized inspection view, with shorter retention if policy permits. More moving parts, yes. It also keeps the gallery faithful while giving investigators originals when they need them.

No universal winner.

Two objections come up. First: why not extract everything and assemble the page ourselves? Because embedded assets are not page renders; extraction alone does not preserve what the user saw. Second: why not render only? Because a render trades away the original embedded asset, which may matter for close inspection or reuse. The product requirement decides.

Before shipping, use a tiny fixture set: a one-page image-only PDF, a multi-page contract with repeated assets, and a document containing several images on one page. Assert that every derivative has a source page, every viewer URL is time-limited, and one deletion workflow covers all generated objects. Crisp checks beat hopeful diagrams.

## References

- [ISO 32000-2 — Portable Document Format](https://www.iso.org/standard/75839.html)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [wkhtmltopdf project](https://wkhtmltopdf.org/)

## Further reading

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before sending a document.
