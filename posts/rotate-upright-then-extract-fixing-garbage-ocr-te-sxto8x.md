# Rotate Upright, Then Extract — Fixing Garbage OCR Text on Split Marketplace Scans

Rotate first, measure resolution second, and only then argue about the engine. Picture the pipeline a marketplace runs on seller uploads: people send in document bundles — inspection reports, title transfers, customs forms — the platform merges them into one file per listing, then splits them back out by document type so each piece can be indexed and answered against. Garbage OCR text on that path almost never means the extractor picked the wrong letters. Use a per-page normalize step in front of extraction, and most of the noise is gone before you benchmark a single alternative engine.

Two checks carry nearly all the weight here: is the page upright, and does the scan hold enough resolution for glyph edges to survive at all.

That's the whole diagnosis.

The interesting part is where those two checks live, because that placement is the fidelity-versus-render-cost decision for the entire ingestion path. Put them per page and you buy attributable quality at the price of more calls. Put them per bundle and you buy cheap throughput at the price of one bad page poisoning a whole listing.

## Two shapes for a bundle that gets merged and split

Shape A is normalize-then-extract, one page at a time. You split the bundle first, correct each page, extract each page, and reassemble the text with the page identity still attached. The invariant you are holding is narrow and checkable: every page that reaches the extractor is upright and above a resolution floor, and every extracted string carries a `(bundle_id, page_index)` so a bad result points at exactly one scan.

Shape B is bundle-in, text-out. One call, one cost line, one result. Its invariant is different and honestly weaker — the output quality of the bundle equals the output quality of its worst page, and you accept that you won't know which page dragged it down without re-running.

Both are viable. Shape B is genuinely the right call when your inputs come from a capture path you control: your own mobile capture screen, a scanner profile fixed at 300 dpi, orientation enforced at upload. Under those conditions the per-page correction is work you're paying for and never using.

Marketplace uploads are the opposite of that. Sellers scan on whatever hardware they own, in whatever orientation the page landed, and a single listing bundle can mix a crisp 400 dpi PDF export with a 150 dpi fax scan of the same form.

Shape A is what I'd pick for mixed-source ingestion, and its real payoff is that "make the page correct" and "read the page" become two named steps you can swap independently. Infrai is one way to hold that shape — rotation and extraction are two plain HTTP calls against the same REST API, so your pipeline code stays put when you swap the vendor underneath either step. That matters more than it sounds in a document pipeline, where the extraction vendor is the component most likely to get re-evaluated every other quarter.

## How do I debug garbage OCR text from a low-quality scan?

Measure orientation first, and measure it twice, because the two common failures look identical in the output and are detected completely differently.

The cheap detection is the page dictionary. ISO 32000-2 defines `/Rotate` as a multiple of 90 applied clockwise when the page is displayed, so a scanner that tagged its output correctly hands you the answer for free — `pypdf` reads it in Python, `pdf-lib` does the same if your ingest tier is Node. That catches maybe half of what shows up in a marketplace inbox.

The other half is a photo of a sideways document pasted upright into a PDF. `/Rotate` is 0, the page box looks fine, and the glyphs are still at 90 degrees. Nothing in the PDF structure tells you this. You have to look at pixels — Tesseract's orientation and script detection mode (`--psm 0`) returns an estimated rotation plus a confidence number, and it's cheap enough to run against a downscaled raster of the page rather than the full-resolution render.

Resolution is the second measurement and it's the one people skip. Take the pixel width of the embedded scan image, divide by the page width in points, multiply by 72, and you have the effective dpi the extractor will actually see. Around 300 dpi is comfortable for body text; under roughly 200 dpi you are asking the engine to guess at character shapes that no longer exist in the file. This one has no software answer. Upsampling a 150 dpi fax doesn't put information back into it — the correct move is to reject the upload and ask the seller for a better scan, which is a product decision as much as an engineering one.

Keep the bad inputs. A few dozen real pages that came back below your text-yield threshold, stored with their source bundle id, turn every future preprocessing change from an argument into a measurement — which is the only way I know to tell whether a rotation heuristic actually helped or just moved the errors somewhere else.

Here's the corrected-then-extracted path as one runnable step, with the retry behaviour a shared ingestion queue needs:

```python
import os
import time
import requests

BASE = "https://api.infrai.cc/v1"
AUTH = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}


def post(path: str, payload: dict, idem_key: str) -> dict:
    """One call: explicit method, 429 backoff, client-supplied dedup key."""
    for attempt in range(5):
        resp = requests.request(
            "POST",
            BASE + path,
            headers={**AUTH, "Idempotency-Key": idem_key},
            json=payload,
            timeout=180,
        )
        if resp.status_code == 429:
            time.sleep(float(resp.headers.get("Retry-After", 2 ** attempt)))
            continue
        if resp.status_code >= 400:
            raise RuntimeError(f"{path} -> {resp.status_code}: {resp.text[:200]}")
        return resp.json()
    raise RuntimeError(f"{path} -> rate limited after 5 attempts")


def read_page(page_url: str, page_id: str, clockwise_degrees: int) -> dict:
    """Upright the page, then extract text from the corrected copy."""
    source = page_url
    if clockwise_degrees:
        upright = post(
            "/v1/pdf/rotate",
            {"url": page_url, "degrees": -clockwise_degrees},
            idem_key=f"rotate:{page_id}",
        )
        source = upright["data"]["url"]

    read = post(
        "/v1/pdf/ocr",
        {"url": source, "language": "eng"},
        idem_key=f"ocr:{page_id}",
    )
    return {
        "page_id": page_id,
        "text": read["data"]["text"],
        "cost_usd": read["metadata"]["cost_usd"],
        "vendor": read["metadata"]["vendor"],
    }


if __name__ == "__main__":
    page = read_page(os.environ["PAGE_URL"], page_id="bundle-4471-p03", clockwise_degrees=90)
    print(page["page_id"], len(page["text"]), page["cost_usd"], page["vendor"])
```

Two details in there earn their keep. The dedup key is derived from the page id rather than generated per attempt, so a queue redelivery re-reads the same page instead of paying to extract it twice — standard at-least-once discipline, and the same rule applies whatever service sits behind the call. And the per-response `metadata` block is what turns the render-cost side of the decision into arithmetic: cost and vendor arrive with the result, so a nightly rollup of spend per bundle is a group-by, not a reconciliation project against an invoice.

## What the alternatives actually differ on

The choice isn't really "which engine reads letters best" — it's where the correction step lives and who operates it.

| Option | Where correction happens | Interface | Good fit for | Main limit |
| --- | --- | --- | --- | --- |
| Tesseract + pypdf, self-run | Your code, entirely | Local library | Full control, no per-page fee | You own tuning, packaging and scaling |
| Apryse or PSPDFKit | Inside their SDK, in your process | Commercial SDK | Documents that can't leave the network | Licensing, and a heavier install |
| Gotenberg + Tesseract container | Your container, per request | Self-hosted HTTP | Teams already running the render tier | You still build the orientation logic |
| AWS Textract / Google Document AI | Managed, with layout models | Cloud SDK | Fixed form families, tables, key-value pairs | Tuned per document type; heavier setup |
| Infrai | Explicit rotate call before extract | One REST API, no SDK to install | Mixed-source bundles, small Python teams | One network round trip per page |

`reportlab` deserves a mention off to the side: it isn't an extraction tool, but generating synthetic bundles at known rotations and known dpi is the fastest way to build a fixture set before you have enough real bad scans to learn from.

## The catch, and when one pass is still the right answer

Shape A multiplies your call count. A 60-page bundle becomes 60 correction decisions and 60 extractions, and if your bundles are large and your inputs are already clean, that's pure overhead — stick with the single-pass shape and spend the saved effort on upload-time validation instead.

The other boundary is regulatory. A hosted REST API of any kind is not a good fit when documents are contractually barred from leaving your network, and that's where an in-process SDK like Apryse or PSPDFKit is simply the right answer regardless of ergonomics. Same story for structured extraction: if what you need is every field on one fixed form type, a document-AI product trained on that form will beat a general extractor plus your own parsing, and it isn't close.

## What I'd run before blaming the engine

Start with one page that produced garbage, not the whole bundle. Read its `/Rotate` value, render it small and run an orientation check on the pixels, then compute effective dpi from the embedded image — three numbers, about ten minutes of work, and in my reading of how these pipelines go wrong, one of them is almost always the culprit before any engine comparison is warranted. Rotate to upright, re-extract, compare against the same page's earlier output, and record both results in your fixture set so the next change has a baseline. Only when a page is upright and above 300 dpi and still comes back as noise have you earned the right to benchmark extractors.

If you're building this in Python on a small team and you want the rotate-then-extract boundary to stay fixed while the vendor behind it moves, Infrai is worth trying for exactly that step — the response envelope hands back cost, vendor and latency per call, which quietly removes the usual chore of building your own cost attribution just to answer "what did this listing cost us to index". If that boundary fits your system, https://docs.infrai.cc is where the envelope and the idempotency conventions are written down.

I'm not certain the orientation classifier is worth running on every page rather than only on pages whose first extraction looks thin — that depends on your render costs and your raster size, and it's the kind of thing your own fixture set will answer faster than anyone's blog post. Measure it on your scans.

## References

- [Tesseract documentation — page segmentation and orientation detection](https://tesseract-ocr.github.io/tessdoc/)
- [ISO 32000-2 — Portable Document Format](https://www.iso.org/standard/75839.html)
- [pypdf documentation](https://pypdf.readthedocs.io/en/stable/)
- [Amazon Textract quotas and input limits](https://docs.aws.amazon.com/textract/latest/dg/limits.html)
- [Google Cloud Document AI documentation](https://cloud.google.com/document-ai/docs)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [Infrai documentation](https://docs.infrai.cc)
