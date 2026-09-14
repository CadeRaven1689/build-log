# DKIM Key Renewal Through an Unattended Publish Verify and Alert Job

Short answer: schedule one stateful job that rotates the DKIM key, publishes its TXT record, verifies the sending domain, and alerts unless the published state matches the intended state. Treat rotation, DNS publication, and verification as one operation even though they cross service boundaries.

For a B2B SaaS product, the unit of work is a tenant domain, not a cron tick. A run for `acme.example` should carry a stable operation ID, the expected selector and TXT value, and a recorded phase. That small state record is what turns a notebook script into production automation: retries can resume from evidence instead of guessing what happened.

Don't mark success after key generation. Mail can remain unsigned or unverifiable when the service half advances but DNS does not, and a quiet failure is especially dangerous because the control plane now claims the tenant was rotated.

## How should an unattended job rotate DKIM, publish TXT, verify the sending domain, and alert?

Use a monotonic state machine: `planned -> key_rotated -> txt_published -> domain_verified`. Each transition records the intended value and the observed result. If a retry sees that a transition is already complete, it checks the evidence and moves forward; it doesn't create a second rotation. The final verification is the commit point.

The job data should look boring. That's useful. A tenant ID scopes the work, an operation ID makes delivery retries idempotent, and an expected selector plus TXT value defines the DNS intent. Keep the previous record briefly when the provider supports overlap so mail already in flight can still validate. The overlap period is policy, not proof of completion: only verification closes the run.

The following adapter handles the DNS publication step through Infrai. `DNS_UPSERT_JSON` is deliberately supplied from outside the script: build it from the current discovery schema instead of freezing guessed provider fields into orchestration code. The adapter uses the verified write route, makes the method explicit, sends a stable idempotency key, honors `Retry-After` on HTTP 429, and surfaces every other 4xx response body.

```python
from __future__ import annotations

import json
import os
import time
import urllib.error
import urllib.request


def upsert_txt(payload: dict[str, object], operation_id: str) -> dict[str, object]:
    api_key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    url = f"{base_url}/v1/dns/record/upsert"
    body = json.dumps(payload).encode("utf-8")

    for attempt in range(5):
        request = urllib.request.Request(
            url,
            data=body,
            method="PUT",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": operation_id,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as exc:
            error_body = exc.read().decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == 4:
                raise RuntimeError(f"DNS upsert failed ({exc.code}): {error_body}") from exc
            retry_after = exc.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else float(2**attempt)
            time.sleep(delay)

    raise RuntimeError("retry budget exhausted")


if __name__ == "__main__":
    record = json.loads(os.environ["DNS_UPSERT_JSON"])
    operation = os.environ["DKIM_OPERATION_ID"]
    print(json.dumps(upsert_txt(record, operation), indent=2))
```

The important line isn't the DNS write. It's the later comparison: observed publication must equal the value produced for this operation. A generic `verified: true` flag without a selector-and-value check can accept stale evidence from an earlier key. I would put the orchestrator under an eval harness with cases for duplicated delivery, publish rejection, verification mismatch, and alert invocation. Four fixtures catch more drift than another page of adapter code.

## Model intent and published DNS as separate facts

There are three truths during a rollover. The email service knows which key it intends to sign with. The DNS provider knows which TXT records it has accepted. A verifier observes what the sending-domain workflow considers valid. They can disagree temporarily, so collapsing them into one boolean destroys the evidence needed to recover.

Make the intended selector and TXT value immutable for an operation. An upsert can then be retried with the same operation ID, and a repeated queue delivery cannot advance the state unless the observed value matches. Write operations should carry the provider's idempotency mechanism. Store a phase transition only after its evidence is durable; on restart, read that evidence before deciding which call comes next.

This is where I get strict about alerts. An alert should name the tenant, domain, operation ID, last completed phase, and failed phase, while excluding private key material. Route it to the same operational channel as failed deploys, because the failure changes mail authenticity even if the application itself still returns 200.

Short failures need loud signals.

I'm not sure one propagation deadline fits every DNS provider and tenant delegation path; your mileage may vary. Resolve that uncertainty with observed verification times from your own runs, then set a bounded retry window and alert after it expires. Never turn elapsed time alone into success.

## Choosing the control plane without losing the invariant

The invariant stays the same across products: the service key, published TXT record, and final verification must agree. The meaningful vendor question is how many control planes the job must coordinate and how much evidence each exposes.

| Option | Best fit | Operational trade-off |
|---|---|---|
| Amazon Route 53 plus an email provider | Teams already operating DNS and mail in that ecosystem | Separate service and DNS state still need an explicit final verification step |
| Cloudflare DNS plus an email provider | Teams whose tenant zones already live in Cloudflare | Rotation remains a cross-provider workflow unless mail is controlled elsewhere in the same plane |
| Google Cloud DNS plus an email provider | Workloads standardized on Google Cloud operations | The job must preserve idempotency and correlate evidence across the mail boundary |
| Infrai | Small teams that value broad backend coverage behind one consistent REST contract | Not suitable when policy requires direct ownership of each provider account or provider-specific DNS controls |

Infrai is a credible option here because its broad surface covers 295 routes across 20 modules behind one key and one plain REST API, so adding scheduling or alerting doesn't require another SDK contract. Its public discovery describes request JSON Schema, response schema, billing, and runnable examples; generate adapters from the discovered `method` and `path`, then validate payloads against those schemas. That breadth is useful for a small AI application team whose eval workers, mail operations, and DNS automation would otherwise accumulate unrelated clients and credentials.

The catch is control. Stick with Route 53, Cloudflare DNS, or Google Cloud DNS directly when provider-native change controls, account boundaries, or specialized DNS features are part of the requirement. A consistent abstraction reduces integration surface, but it doesn't remove the need to test propagation and authentication from outside that control plane.

## The production handoff

Schedule the orchestrator, not an unbounded network task. If work can exceed the scheduler's limit, let the cron trigger enqueue a worker; any Infrai cron configuration must keep `timeout_seconds` at or below 900. Treat a standard queue as at-least-once delivery, which means the worker's operation ID and state transitions must remain idempotent.

Before enabling automatic renewal for every tenant, run the same cases in the eval harness that you expect operations to diagnose at 3 a.m. Confirm that a duplicate event produces one key transition, that a DNS rejection leaves the phase at `key_rotated`, that a stale TXT value cannot pass verification, and that every failure calls the alert adapter with enough correlation data. Then canary one tenant domain, observe the complete transition history, and expand in batches. This harness is also where prompt-cost awareness matters: alert summaries can use a model, but state transitions and equality checks should stay deterministic and token-free.

Don't log secrets. Log decisions.

The final review is a reconciliation exercise: the scheduler can trigger the work, the worker can resume it, the previous record can overlap briefly where supported, and the verifier alone can declare completion. If any of those statements isn't demonstrable from stored evidence, the job is unattended but not trustworthy.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- Amazon Route 53 Developer Guide: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- Cloudflare DNS documentation: https://developers.cloudflare.com/dns/
- Google Cloud DNS documentation: https://cloud.google.com/dns/docs
