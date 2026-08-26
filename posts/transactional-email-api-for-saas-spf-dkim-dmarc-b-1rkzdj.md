# Transactional Email API for SaaS: SPF, DKIM, DMARC, Bounces, and Node.js

**Short answer: choose a transactional email API by its deliverability control loop, not its send call.** For a SaaS that doesn't want an SMTP relay, the minimum useful loop is authenticated HTTP sending, verified SPF/DKIM/DMARC alignment, durable delivery events, application-owned bounce suppression, and a recoverable polling cursor. A Node.js client is convenient, but a plain REST contract is easier to evaluate, port, and reproduce from a notebook.

The API accepting a message is only the first state transition. The application should write a correlation ID before sending, store the remote message ID after acceptance, poll events into an idempotent ledger, and suppress recipients only from explicit terminal outcomes. DNS checks and regional data-flow checks belong in the same release gate. That flow is the decision.

Don't start with a feature matrix.

## How can a SaaS trace the delivery loop before comparing APIs?

Begin with one password-reset message in a staging domain. Create an application correlation ID, send through the candidate's HTTP API, and persist the returned message ID beside the template version and recipient. The polling worker then reads from a committed cursor, normalizes remote event names into a small internal vocabulary, inserts unseen events, updates suppression state, and advances the cursor in one database transaction. If the transaction rolls back, the page is fetched again; uniqueness on the remote event ID makes that replay harmless.

This is a deliberately narrow adapter. It keeps provider payloads out of account recovery, billing, and notification code. It also gives an eval harness something stable to assert: one accepted message joins to its later events; two deliveries of the same event create one ledger row; a hard bounce suppresses future sends; a temporary failure does not become a permanent suppression without an explicit policy; and restarting from an old cursor converges on the same state. The notebook version can use fixtures, but the production gate must exercise the real HTTP boundary and DNS.

Here is the core of a polling worker in Python. The candidate-specific URLs, authentication header, event mapping, and pagination fields are configuration because no cross-vendor standard defines them. The same contract can be called from Node.js with its built-in `fetch`; requiring a proprietary SDK would make the evaluation less portable.

```python
from dataclasses import dataclass
from typing import Callable, Iterable


@dataclass(frozen=True)
class DeliveryEvent:
    event_id: str
    message_id: str
    recipient: str
    kind: str
    occurred_at: str


@dataclass(frozen=True)
class EventPage:
    events: tuple[DeliveryEvent, ...]
    next_cursor: str


TERMINAL_SUPPRESSIONS = {"hard_bounce", "complaint"}


def apply_page(
    page: EventPage,
    insert_once: Callable[[DeliveryEvent], bool],
    suppress: Callable[[str, str], None],
    commit_cursor: Callable[[str], None],
) -> None:
    for event in page.events:
        if insert_once(event) and event.kind in TERMINAL_SUPPRESSIONS:
            suppress(event.recipient, event.kind)
    commit_cursor(page.next_cursor)


def drain_events(
    cursor: str,
    fetch_page: Callable[[str], EventPage],
    apply: Callable[[EventPage], None],
    max_pages: int = 100,
) -> str:
    current = cursor
    for _ in range(max_pages):
        page = fetch_page(current)
        apply(page)
        if page.next_cursor == current or not page.events:
            return page.next_cursor
        current = page.next_cursor
    return current
```

`apply_page` still needs a real database transaction around event insertion, suppression, and cursor advancement. The point is ownership: transport-specific decoding happens before this function, while durable delivery meaning stays inside the application. Keep the raw event too. A new normalizer can then replay old input after a mapping change rather than guessing from the reduced state.

Failure policy should be equally explicit. A `401` stops the worker and alerts because replaying with the same credentials won't help. A `429` follows the API's documented retry signal without advancing the cursor. A malformed event goes to a quarantine path with its raw payload and blocks cursor commitment until the team has decided how to classify it. This is where prompt-cost discipline has an analogue: every retry and every stored payload has a cost, so measure them, but don't optimize away the evidence needed to explain a missing recovery email.

## Authentication is a release gate, not a dashboard badge

Use a dedicated transactional subdomain and inventory every sender that already uses the organizational domain before changing policy. SPF identifies authorized sending sources. DKIM attaches a cryptographic signature and signing identity; RFC 6376 describes that mechanism and its verification model. DMARC evaluates alignment with the visible From domain and publishes a handling policy. Passing one check does not substitute for the others, and none of them proves that a recipient wanted the mail.

For each candidate, record the visible From domain, envelope domain, DKIM signing domain, selector rotation procedure, SPF include or other required record, DMARC reporting destination, and the exact evidence shown by a received message. Then test authentication again after rotating a key. DNS propagation makes this rollout environment-dependent — I'm not sure any universal waiting interval is defensible — so the gate should inspect authoritative DNS and received headers rather than sleep for a fixed number of minutes.

Password-reset mail needs application controls beyond deliverability. OWASP recommends consistent responses for existing and nonexistent accounts, comparable response timing, a side channel for delivery, random single-use expiring tokens, rate limiting, and no account change until a valid token is presented. The email API transports that token. It doesn't make the recovery flow safe.

## How should a Node.js SaaS test transactional email API event polling?

Build the eval before choosing the transport. A good fixture suite covers acceptance, delivery, hard bounce, complaint, temporary failure, duplicate event, empty page, cursor replay, pagination, and a worker restart between fetch and commit. The live suite should send only to controlled addresses and verify that the remote message ID can be joined to a later event without parsing a subject line or embedding customer data in tags. One concrete recovery run starts with an empty ledger and a saved cursor, sends several controlled messages, and records every acceptance ID. Pause the poller before it reads any outcome, allow enough events to require more than one page under the candidate's documented pagination behavior, and start it again. Interrupt it once after inserting a page but before cursor commitment, then replay from the saved cursor. The assertions are intentionally unforgiving: every acceptance ID joins to its expected terminal or pending state, each remote event ID appears once, suppression is applied once, and the final cursor is committed only after its page. Run the same fixture twice with shuffled duplicate input. This doesn't estimate inbox placement, but it does prove that an ordinary deployment interruption cannot silently turn a known event into an unknown customer complaint.

Then stop the poller long enough to accumulate several pages and restart it from the last committed cursor. The required result is convergence: no lost events, no duplicate business action, and a cursor whose age returns below the alert threshold. Check the candidate's written event-retention window against the maximum realistic outage of your worker. “Real time” is irrelevant if history disappears before a recovery job can fetch it.

Watch four production signals: age of the last committed cursor, age of the oldest unresolved accepted message, counts by normalized outcome, and suppression attempts prevented before send. Segment outcomes by transactional subdomain and message class, not by individual user in a broad metrics label. High-cardinality recipient labels make dashboards expensive and leak information into places that don't need it.

Polling has a genuine trade-off. It avoids public webhook ingress and makes backfill easy to reason about, but it increases detection latency and spends requests on empty pages. Webhooks can reduce latency, yet they require request authentication, replay defense, and monitoring of the inbound endpoint. If both are used, feed them through the same idempotent normalizer. Two independent delivery ledgers will disagree eventually.

## The selection table should contain disqualifiers

Score evidence from the proof of concept, not promises from a sales page. The first four rows are hard gates for this use case; the remaining rows reveal operational cost and lock-in.

| Criterion | Evidence to require | Reject when |
| --- | --- | --- |
| HTTP send | Documented request, authentication, idempotency behavior, and stable message ID | SMTP is required or acceptance cannot be correlated |
| Domain authentication | SPF instructions, DKIM signing and rotation, DMARC alignment evidence | The visible and signing domains cannot be verified |
| Delivery events | Cursor or stable pagination, event identity, timestamps, and documented retention | Recovery after downtime can lose or reorder state irreparably |
| Suppression | Exportable bounce and complaint outcomes with reason categories | The application cannot prevent a known bad recipient from being retried |
| US/EU boundary | Written processing, storage, subprocessors, transfer mechanism, and region-selection terms | The claimed region covers only the API hostname or dashboard |
| Portability | Plain HTTP contract, raw event access, and export path | Business logic must depend on opaque SDK objects |
| Operations | Rate-limit semantics, credential rotation, audit access, and status communication | A `401` and `429` cannot drive distinct worker policies |

“EU region” is not a complete answer. Ask separately where message bodies, recipient addresses, event logs, suppression lists, backups, and support access are processed and retained. Legal and security reviewers must decide what is acceptable; an engineering note should preserve the vendor's written answers and the date they were checked. Your mileage may vary because the right boundary depends on customer contracts and the data placed in each template.

The catch is that this architecture is not suitable for every team. Stick with an existing SMTP relay when it already has proven authentication, event export, suppression, and regional terms, and the cost of changing application code exceeds the benefit of HTTP. Prefer webhook-first delivery when sub-minute status drives the product and the team already operates authenticated public ingress. A local polling ledger also adds schema, migrations, retention work, and on-call ownership. Outsourcing transport doesn't outsource those decisions.

## Operate the contract you evaluated

Before launch, freeze the normalized event vocabulary, run the replay suite, verify authoritative DNS and received-message authentication, rotate a test credential and DKIM key, test suppression before the send call, and recover from a deliberately stale cursor. Do the same checks from the production runtime so an egress or secret-loading difference cannot hide behind a passing notebook.

After launch, review unresolved-message age and cursor age as service objectives, sample raw-to-normalized mappings, and rehearse export plus transport replacement. Re-check regional and retention terms on a schedule because written policies can change. Keep template revisions and correlation IDs transport-neutral, and keep sensitive reset tokens out of logs and event metadata.

The best transactional email API is the one whose delivery state your SaaS can prove, replay, and replace. Everything else is a send demo.

## Sources

- https://datatracker.ietf.org/doc/html/rfc6376
- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
