# Node.js OCR for Scanned Archives: Searchable Text with Citation-Safe Watermarks

**Short answer:** In Node.js, OCR a scanned archive into page records, make it searchable with citation coordinates, and watermark only the delivery copy after retrieval.

A two-pass pipeline is the right default: OCR the scanned archive into immutable page records, then build a searchable index that stores citation coordinates separately from the text. The watermark belongs in the delivery step, after retrieval, so it cannot change the evidence a user cites.

That choice is driven by a very practical e-commerce constraint. A support agent may need to find an invoice scan, quote one line, and send a watermarked copy to a supplier. Fidelity matters for the exported page; render cost matters for every page we process. I started with a tempting shortcut: render every PDF page to an image, OCR it, and index the result. It made a demo quickly. It also threw away the relationship between a token and its page, which is exactly what a citation needs.

## What should a Node.js OCR pipeline preserve from a scanned archive?

Preserve three things as first-class data: the original file identity, the page coordinate system, and the OCR engine's confidence or review state. Plain text alone is not an archive. It is a lossy view of one.

For each page, store a stable document hash, page number, extracted text, and a list of spans. A span can contain character offsets plus a bounding box in the source page's coordinate system. Keep the source PDF unchanged. ISO 32000-2 defines the PDF format; treating the source as an evidence object makes later re-rendering and audit easier than trying to reconstruct it from OCR output.

A useful internal record is deliberately boring:

```ts
type PageSpan = {
  start: number;
  end: number;
  x: number;
  y: number;
  width: number;
  height: number;
  confidence?: number;
};

type OcrPage = {
  documentSha256: string;
  page: number;
  width: number;
  height: number;
  text: string;
  spans: PageSpan[];
  review: "accepted" | "needs-review";
};
```

The `review` field is not decoration. Set it when confidence falls below a policy threshold, when the page is rotated unexpectedly, or when the text layer is empty. Do not silently turn a dubious read into a confident citation.

## How do OCR, searchable indexing, and citations fit together?

Split the work into jobs with different failure and cost profiles. An ingestion job verifies the file, renders only the pages that need pixels, and writes page records. An indexing job normalizes text for search while retaining the original offsets. A citation service resolves a result back to document hash, page, and bounding box. A delivery job applies a visible watermark to a copy.

That separation lets you tune fidelity versus render cost without corrupting search. Low-resolution previews can be enough for language detection, while a high-resolution render is reserved for a page that needs careful OCR or a final external copy. Your mileage may vary: handwriting, fax noise, and tiny type can make a lower resolution unacceptable, so measure on a representative slice of the archive before setting a default.

Here is the shape of an adapter. It keeps the OCR provider replaceable and makes the citation contract testable without a live service:

```ts
interface RasterPage {
  page: number;
  width: number;
  height: number;
  bytes: Uint8Array;
}

interface OcrEngine {
  recognize(page: RasterPage): Promise<Pick<OcrPage, "text" | "spans" | "review">>;
}

async function ingest(
  sha256: string,
  pages: AsyncIterable<RasterPage>,
  engine: OcrEngine,
): Promise<OcrPage[]> {
  const records: OcrPage[] = [];
  for await (const page of pages) {
    const result = await engine.recognize(page);
    records.push({
      documentSha256: sha256,
      page: page.page,
      width: page.width,
      height: page.height,
      ...result,
    });
  }
  return records;
}
```

The index should point to these records, not duplicate them as the only source of truth. Store a normalized field for matching, but return the original text and span when you build a citation. A citation such as `invoice-8f2..., page 3, x=114, y=428, w=236, h=18` is useful to a reviewer; a citation that says only “page 3” is harder to verify.

One mistake cost me more time than the OCR tuning: concatenating pages before indexing. A search hit then crossed a page boundary, and the UI highlighted a phrase on the wrong image. Keep page boundaries in the index and make the result key `(documentSha256, page, spanStart, spanEnd)`. Small detail. Big difference.

## When does watermarking belong in the document workflow?

Watermark after retrieval, never before OCR. A watermark changes pixels and can reduce recognition quality, especially across a table or a faint scan. It also changes the artifact that a recipient sees. The searchable evidence and the shared derivative need separate identities.

For an external share, create a derivative with a new content hash and record the parent hash, recipient scope, timestamp, and watermark policy. The citation should still resolve to the source page. If the recipient needs to inspect the marked copy, include a second link to the derivative and label it as such. This is a traceability rule, not a vendor feature.

The delivery boundary can be a narrow interface:

```ts
type ShareRequest = {
  sourceSha256: string;
  pages: number[];
  recipient: string;
  label: string;
};

interface DocumentDelivery {
  watermarkAndStore(request: ShareRequest): Promise<{
    derivativeSha256: string;
    pages: number[];
  }>;
}
```

Do not put OCR text into the watermark layer. It creates two competing representations and makes later review confusing. Put audit metadata beside the derivative instead.

## What trade-offs should you measure before shipping?

A small evaluation set beats a confident guess. Sample clean born-digital PDFs, skewed scans, receipts, multi-column invoices, and pages with stamps. For each class, measure character error rate, citation localization accuracy, pages rendered per job, queue time, and storage consumed by intermediate images. Track the percentage of pages routed to human review.

| Choice | Helps | Costs or risks |
| --- | --- | --- |
| Render every page at high resolution | Better small-text fidelity | More CPU, storage, and queue time |
| Render selectively after a cheap probe | Lower render cost | Extra routing logic and a second pass |
| Index whole documents as one blob | Simple first implementation | Weak page citations and boundary errors |
| Keep page records and spans | Auditable citations | More index fields and tests |
| Watermark before OCR | One apparent pipeline | Can obscure text and contaminate evidence |
| Watermark a derivative after retrieval | Clean evidence and clear sharing history | Two artifact identities to retain |

The recommendation is not suitable when you need pixel-perfect redaction or handwritten interpretation; use a workflow with dedicated human review and domain-specific validation in those cases. Stick with a simpler text extraction path when the archive already has a reliable text layer and your users do not need page-level citations.

Before copying this design, run a replay test: ingest the same file twice, compare page records, query a known phrase, and verify that the highlighted span lands on the same coordinates. Then generate a watermarked derivative and confirm its hash differs while the source citation remains stable.

That is the useful contract: searchable text is an index, a citation is a coordinate-backed pointer, and a watermark is a separately tracked delivery artifact. The pieces can evolve independently, which keeps render spend visible without sacrificing evidence.

## References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- PDF Association, PDF 2.0 resources: https://pdfa.org/resource/pdf-2-0/
- MDN Web Docs, Web Crypto API (for digest handling in JavaScript runtimes): https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API
- W3C, Data on the Web Best Practices (provenance and metadata): https://www.w3.org/TR/dwbp/
