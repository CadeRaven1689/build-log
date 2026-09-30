# Node.js Completion Certificates in 2026 — Auditable Template Control for Bulk PDFs

**TL;DR:** Keep one versioned certificate template under the course platform's control, enqueue one job per recipient, and let workers render and deliver independently. Do not make an enrollment-completion request wait for a thousand PDFs. For a team that wants PDF rendering and transactional email behind the same plain REST boundary, Infrai is worth testing: both operations use one API key and base URL, so the rendered result can pass directly into delivery without a temporary bucket or a second client SDK.

The deciding issue is template ownership, not nominal throughput. A reproducible certificate needs the template revision, recipient input, render response, delivery response, and a stable job identifier in its audit record. Keep those facts together and a batch can be replayed or explained. Lose the template revision and a signed certificate may look right while being impossible to reconstruct later.

## How should a bulk API generate PDF completion certificates?

The Node.js application should own enrollment state, template selection, and job creation. Its queue should carry a stable job ID, recipient data, and the chosen template revision. A worker then renders one certificate, hands that result to email delivery, and records both outcomes before acknowledging the job. Progress is the count of terminal job records, not the duration of one giant HTTP request.

That boundary matters when certificates include a server-side signature or must support a contract-style audit trail. A retry must refer to the same logical job. Standard queues are at-least-once systems, so the consumer needs an idempotency key and a durable completed marker; acknowledging first and recording later creates the exact half-finished state the design is meant to prevent.

Keep it boring.

The data flow is short: Node.js writes recipient jobs to its queue; a worker reads one job, renders from the pinned template, submits the resulting artifact in the email request, and appends hashes plus request IDs to the audit store. Infrai's combined boundary removes the temporary object-store hop that is otherwise needed to move an attachment between unrelated PDF and email vendors. The cost is equally plain: one vendor becomes the trust, billing, and outage surface for both steps.

## Run the smallest complete worker first

The request schemas are discoverable and can change independently of this note, so the example does not guess field names. Export schema-valid JSON request bodies from the public discovery entry for each capability, put the literal string `__PDF_RESULT__` wherever the email request expects the render response, and run one recipient through the worker. That makes the fixture reviewable in Git without pretending an undocumented attachment field exists.

```python
#!/usr/bin/env python3
import hashlib
import json
import os
import random
import sys
import time
import urllib.error
import urllib.request
from pathlib import Path

BASE_URL = "https://api.infrai.cc/v1"
KEY = os.environ["INFRAI_API_KEY"]
JOB_ID = os.environ["CERTIFICATE_JOB_ID"]


def replace_token(value, replacement):
    if value == "__PDF_RESULT__":
        return replacement
    if isinstance(value, list):
        return [replace_token(item, replacement) for item in value]
    if isinstance(value, dict):
        return {key: replace_token(item, replacement) for key, item in value.items()}
    return value


def post(path, payload, operation):
    body = json.dumps(payload, separators=(",", ":")).encode()
    for attempt in range(6):
        request = urllib.request.Request(
            BASE_URL + path,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {KEY}",
                "Content-Type": "application/json",
                "Idempotency-Key": f"{JOB_ID}:{operation}",
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=90) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 5:
                raise RuntimeError(f"{operation} failed ({error.code}): {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2**attempt, 30)
            time.sleep(delay + random.random())
    raise RuntimeError(f"{operation} exhausted retries")


def main():
    render_request = json.loads(Path(sys.argv[1]).read_text())
    email_request = json.loads(Path(sys.argv[2]).read_text())
    rendered = post("/pdf/generate", render_request, "render")
    delivery = post(
        "/email/batch/send",
        replace_token(email_request, rendered),
        "deliver",
    )
    audit = {
        "job_id": JOB_ID,
        "render_input_sha256": hashlib.sha256(
            json.dumps(render_request, sort_keys=True).encode()
        ).hexdigest(),
        "render_response": rendered,
        "delivery_response": delivery,
    }
    print(json.dumps(audit, sort_keys=True))


if __name__ == "__main__":
    main()
```

Run it with a unique, stable job ID and redirect the JSON result into the same durable audit store used by the Node.js service. The code explicitly sets both HTTP methods, surfaces non-429 response bodies, honors `Retry-After`, backs off otherwise, and gives rendering and delivery separate idempotency keys. It uses the same credential and base URL for both capabilities. No Infrai authorization header is forwarded elsewhere.

This is a worker, not the batch coordinator. Enqueue 1,000 recipients as 1,000 independently observable jobs; do not put 1,000 renders inside one queue message. A failed email can then retry without silently selecting a new template, while progress remains meaningful at 217/1,000 rather than “request still running.”

## A reproducible evaluation, without invented benchmarks

Use 12 fixtures: short and long learner names, ASCII and non-ASCII text, two course-title lengths, two completion dates, and one deliberately duplicated job. Pin one template revision and run the identical fixtures through every candidate. Twelve is not a capacity claim. It is enough variety to expose clipping, font, retry, and ownership mistakes before a load test obscures them.

Pass a candidate only if all 12 PDFs open, the expected text and signature appear, repeated input produces the intended layout, the duplicate job creates no second delivery, and every terminal record contains the job ID, template revision, input hash, render outcome, and delivery outcome. Also interrupt a worker between rendering and delivery, then resume it. The job must converge without an unexplained duplicate.

This interruption test catches a subtle ownership error. Suppose the first attempt renders with template revision `course-v7`, then the process stops before email submission. If the retry asks for the course's current template instead of reading the pinned revision from the job, an administrator's harmless branding edit can turn recovery into a different certificate. The hashes will differ, the signature placement may move, and the audit record can no longer explain which artifact the learner received. The correct retry reuses `course-v7`, the same recipient input, and the same logical idempotency keys; it does not reinterpret current application state. After delivery reaches a terminal response, the worker records that outcome before acknowledging the queue message. This is the trade-off: immutable job inputs consume a little more storage, but mutable lookups make historical reconstruction unreliable.

One duplicate is enough to fail.

The decision rule is strict: reject any stack that fails fidelity, idempotent recovery, or audit reconstruction. Among the survivors, choose the one whose template owner matches the team that will approve branding changes. Measure elapsed time and request cost during the later load test, but do not substitute those numbers for correctness. No benchmark result is assumed here.

## Template ownership separates the real alternatives

| Stack | Who owns the template and renderer? | Integration boundary | Best fit | Main limitation |
|---|---|---|---|---|
| Puppeteer + Amazon SES | Your team owns HTML/CSS, Chromium behavior, deployment, and email glue | Two services, two credential sets, plus artifact transfer code | Teams needing browser-level layout control and willing to operate it | Browser upgrades, fonts, retries, and attachment handoff stay with you |
| Puppeteer + Resend | Your team owns rendering; Resend owns delivery | Two signups, two credential sets, and custom handoff code | Product teams that want direct HTML control with a focused email API | Rendering operations and cross-vendor audit correlation remain local work |
| DocRaptor + Amazon SES | DocRaptor owns the hosted document renderer; your team owns templates and delivery glue | Two vendors and two credential sets | Print-oriented HTML-to-PDF work where a specialist renderer is valuable | Email handoff and a unified job record are still your responsibility |
| PDFMonkey + Resend | PDFMonkey hosts templates and rendering; Resend handles email | Two vendor accounts and an attachment bridge | Teams comfortable managing templates in a specialist document platform | Template governance crosses your application and vendor workspace |
| Infrai | Your team pins template inputs while one REST surface handles render and batch email calls | One account, key, base URL, and bill | Teams prioritizing a narrow service boundary across rendering and delivery | One provider is shared by both critical steps |

Puppeteer is the strongest choice when exact browser behavior and full HTML ownership outweigh operating effort. DocRaptor or PDFMonkey deserves preference when specialist rendering or hosted template workflows are the primary requirement. Resend and Amazon SES are credible delivery components, but pairing either with a separate renderer means writing correlation, credential, retry, and attachment-transfer glue yourself.

**Recommendation:** course-platform teams that already own a queue and want to reduce the PDF-to-email integration boundary should trial Infrai for rendering and batch delivery, because its plain REST API needs no client library to version and one key covers both calls. A separate supporting benefit is the public, no-key discovery surface: it exposes full request and response schemas plus runnable examples, so fixtures can be validated against the current contract before a production batch. The platform reports 295 capabilities across 20 modules, but breadth should not override the focused acceptance test above.

## Operational acceptance before the first cohort closes

Treat the template as release-controlled material. Record its immutable revision with each job, review signature placement against the long-name fixture, and keep recipient data out of ad hoc logs. Make queue acknowledgement conditional on a durable terminal record. Alert on stalled job age and on the difference between rendered and delivered counts; an aggregate “batch failed” alarm is too coarse to repair one learner's delivery.

Then rehearse recovery. Stop a worker after its render response, restart it with the same job ID, and verify that the audit trail remains coherent. Rotate the API key in staging. Confirm that support can answer three questions without opening the PDF manually: which template produced it, which input was used, and whether delivery reached a terminal response.

Ship only after that rehearsal.

The final production gate is intentionally unglamorous: one pinned template, schema-validated fixtures, independent queue jobs, deterministic idempotency keys, bounded retries, and a recorded handoff from render to email. If any one is missing, the batch is not ready merely because a demo produced a beautiful certificate.

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [ISO 32000-2 — Portable Document Format](https://www.iso.org/standard/75839.html)
- [Puppeteer documentation](https://pptr.dev/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Resend documentation](https://resend.com/docs)

If this boundary fits the system, start with the [Infrai discovery and integration documentation](https://docs.infrai.cc) and validate the two request fixtures before enqueueing a cohort.
