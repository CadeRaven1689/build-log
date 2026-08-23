# Cheapest Speech to Text API: Per-Minute Pricing Comparison for EU Startups

---

Short answer: there is no durable cheapest speech-to-text API until you evaluate the same audio sample, language mix, output requirements, and current per-minute quote against your own error budget. For a startup transcribing audio to text, structured output correctness matters more than a small headline price difference: a transcript that loses invoice numbers or line-item boundaries can cost more to repair than it saved.

The practical test is small and unglamorous: take a fixed set of supplier recordings, transcribe them, extract invoice fields, and count the fields that a human accepts without editing. A transcript can look fluent while turning `1,018.50` into `1018.50`, dropping a decimal, or joining a supplier name to the next line. Structured output correctness is the decision axis. Price is only one input.

## How can a startup keep speech-to-text API invoices accurate in the EU?

Start with a worksheet whose rows are real audio cases: short phone recordings, noisy warehouse notes, accented speech, mixed languages, and an invoice number read aloud. Keep the audio duration fixed. Ask every candidate for the same output fields and record both usage and correction work.

A useful cost model is:

```python
from dataclasses import dataclass


@dataclass
class CandidateResult:
    audio_minutes: float
    api_cost: float
    accepted_fields: int
    total_fields: int
    correction_minutes: float


def cost_per_accepted_field(result: CandidateResult) -> float:
    accepted = max(result.accepted_fields, 1)
    labor_cost = result.correction_minutes * 0.0  # Supply your local labor rate.
    return (result.api_cost + labor_cost) / accepted
```

The zero in that example is intentional: insert the team's actual correction rate instead of pretending it is universal. For a first pass, keep API cost and correction time as separate columns. Then add storage, retries, and any second transcription pass. A provider that costs less per minute but needs many field repairs may lose on the metric that matters.

Do not copy a price table from an old comparison. Pricing changes, regional taxes can be separate, and a public page may distinguish prerecorded audio from streaming. I'm not sure any static article can answer the word “cheapest” for every startup; a dated fixture and a current quote can.

## What EU data and retention gates should a startup set before choosing an API?

Speech recognition is an intermediate artifact. The application needs a typed record, and that record needs validation. “Total: one thousand eighty euros and fifty cents” is not enough if the parser emits a string with an ambiguous decimal separator. “Invoice number” is not enough if a leading zero disappears.

Treat the pipeline as four contracts:

1. Audio intake records duration, MIME type, locale, and an idempotency key.
2. Transcription returns text plus timestamps or confidence data when the chosen service provides them.
3. Field extraction maps text into a strict schema such as supplier, invoice number, currency, subtotal, tax, and total.
4. Validation checks arithmetic, currency, required fields, and human-review thresholds.

The extraction stage should not quietly repair an impossible result. If subtotal plus tax does not equal total within the business rule, route the document for review and preserve the original transcript. That distinction makes evaluation honest: the system can be wrong in a visible way instead of presenting a polished but unsafe answer.

A small Python boundary keeps vendor-specific code away from the business rules:

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass
class Transcript:
    text: str
    locale: str


class SpeechToText(Protocol):
    def transcribe(self, audio_bytes: bytes, *, locale: str) -> Transcript:
        ...


def review_invoice(transcript: Transcript) -> str:
    if not transcript.text.strip():
        return "human_review"
    return "extract_fields"
```

The interface is deliberately boring. Good. It lets an eval harness compare providers without rewriting invoice validation or deployment code.

## How should the evaluation harness measure accepted fields?

Build a labeled set before you negotiate on price. For each recording, store the expected field values, acceptable normalizations, and a reason for human review. Measure exact match for invoice number and currency, numeric tolerance for totals, and a separate score for supplier-name normalization. A single aggregate accuracy score hides the expensive failures.

I use a 42-row fixture as a useful starting shape for a local harness, then expand it when a new language, microphone, or supplier format arrives. That number is a test-set design choice, not a benchmark claim. The harness should report at least:

- field-level exact-match rate;
- document-level all-required-fields rate;
- false acceptance rate for values that pass validation but are wrong;
- review rate and correction minutes;
- billed audio minutes, retry count, and end-to-end latency.

Log the model or service version, request timestamp, locale, audio duration, and schema version. Store redacted examples where invoices contain personal or commercial data. In the EU, make retention and processing location explicit in the team's compliance review; do not infer them from a marketing label.

A failure with a 400-style validation response should be distinguishable from a transcription mismatch, a timeout, and a downstream schema error. Those are different fixes. The dashboard should preserve that distinction, with a request ID carried across intake, transcription, extraction, and review.

## When should a team keep a more expensive option?

The catch is operational fit. A low per-minute rate is not suitable when the service lacks a required language, cannot meet the application's latency target, has a retention policy the team cannot accept, or produces too many high-cost numeric mistakes. Stick with a more expensive option when its measured accepted-field rate materially reduces review work and the business can justify the difference.

The reverse is also true. A premium feature is wasted when recordings are clean, the language set is narrow, review is cheap, and a simpler provider clears the correctness threshold. The decision rule should be explicit: reject any candidate below the minimum document-level correctness target, then choose among the survivors using total cost, latency, operational burden, and policy fit.

That gives a startup a defensible answer to the pricing question without turning a volatile number into a promise. Run the fixture again when rates, models, locales, or invoice formats change. Short list first.

## References

- https://github.com/openai/tiktoken
- https://python.langchain.com/docs/integrations/chat/openai/
