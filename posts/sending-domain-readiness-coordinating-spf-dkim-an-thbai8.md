# Sending-Domain Readiness: Coordinating SPF, DKIM, and DMARC with Verifiable Evidence

TL;DR: Treat sending-domain setup as one idempotent reconciliation job, but do not pretend three DNS changes are a transaction. Store one desired record set, upsert each logical TXT record, read all three back through DNS, and activate the domain only when the observed values match that intent. For a B2B SaaS admin console, the useful output is evidence: which owner name and value were observed, when they were checked, and why activation is still blocked.

A successful write response is not proof that a later DNS read returns the intended data. SPF, DKIM, and DMARC also occupy different owner names, so one opaque `verified` boolean throws away the detail an operator needs. **One job should own the intent; each record should retain its own evidence.**

## How should one job publish SPF, DKIM, and DMARC TXT records?

The job should guarantee convergence, not simultaneous publication. Its input is a versioned desired state for one sending domain: the SPF policy at the domain, a DKIM public key under a selector, and the DMARC policy under `_dmarc`. A stable digest of that input identifies the revision. Replaying the same revision produces the same writes and verification criteria.

The data flow is compact. The admin console submits desired state, a worker reconciles it through a DNS adapter, a separate read path observes logical TXT values, and an activation gate evaluates the evidence. Persist the desired-state digest and per-record observations beside the tenant and domain. That makes retries boring, which is exactly what a notebook-to-production path needs.

Retries happen.

Do not derive the idempotency key from a timestamp or attempt number. Those identify execution, not intent. Serialize work by sending domain too. Consider revision A entering the queue, revision B being submitted a moment later, and the A worker resuming after B has completed: without a compare-and-set against the stored current revision, A can publish old values and then record misleading evidence against them. The guard belongs immediately before each write and again before activation, because a worker can lose its lease between those points. Revision B should win because it is current, not because its worker happened to finish last. This is a real trade-off in the state machine: extra reads and conditional writes cost work, but they prevent an old, perfectly repeatable job from becoming an idempotent rollback.

## A runnable reconciler before the trade-offs

This example uses an in-memory adapter, so its control flow runs without credentials. A production adapter needs the same two operations: replace the application-owned logical TXT value for an owner name, then read the logical TXT values visible through the verification path.

```python
from __future__ import annotations

from dataclasses import dataclass
from hashlib import sha256
import json
from typing import Protocol


@dataclass(frozen=True)
class TxtIntent:
    owner: str
    value: str


class DnsAdapter(Protocol):
    def replace_txt(self, owner: str, value: str) -> None: ...
    def read_txt(self, owner: str) -> list[str]: ...


class MemoryDns:
    def __init__(self) -> None:
        self.records: dict[str, list[str]] = {}

    def replace_txt(self, owner: str, value: str) -> None:
        self.records[owner] = [value]

    def read_txt(self, owner: str) -> list[str]:
        return list(self.records.get(owner, []))


def desired_records(domain: str, selector: str, public_key: str) -> list[TxtIntent]:
    return [
        TxtIntent(domain, "v=spf1 include:_spf.example.net -all"),
        TxtIntent(
            f"{selector}._domainkey.{domain}",
            f"v=DKIM1; k=rsa; p={public_key}",
        ),
        TxtIntent(
            f"_dmarc.{domain}",
            f"v=DMARC1; p=none; rua=mailto:dmarc-reports@{domain}",
        ),
    ]


def revision(records: list[TxtIntent]) -> str:
    payload = [{"owner": item.owner, "value": item.value} for item in records]
    encoded = json.dumps(payload, sort_keys=True, separators=(",", ":")).encode()
    return sha256(encoded).hexdigest()


def reconcile(adapter: DnsAdapter, records: list[TxtIntent]) -> dict[str, object]:
    for item in records:
        adapter.replace_txt(item.owner, item.value)

    observations: dict[str, dict[str, object]] = {}
    for item in records:
        observed = [value.strip() for value in adapter.read_txt(item.owner)]
        observations[item.owner] = {
            "expected": item.value,
            "observed": observed,
            "matched": item.value in observed,
        }

    return {
        "revision": revision(records),
        "ready": all(item["matched"] for item in observations.values()),
        "observations": observations,
    }


records = desired_records(
    domain="mail.customer.example",
    selector="saas1",
    public_key="BASE64_PUBLIC_KEY",
)
dns = MemoryDns()
result = reconcile(dns, records)
assert result["ready"] is True
print(result["revision"])
```

The placeholder SPF include target and DKIM key must come from the real sending configuration; they make no claim about a service. The DMARC reporting mailbox must exist operationally before reports are useful. More important, `replace_txt` means replace the application-owned logical record, not delete every TXT value at the owner name. Other TXT data may belong to another system.

There is a sharp edge in the sample's convenience: a real DNS response can present a TXT value as multiple character strings. The adapter should join the strings belonging to one resource record before returning a logical value. Keep that parsing below the interface so the gate compares policies, not presentation details.

## Why is a successful write insufficient?

A control-plane acknowledgement establishes only that the write endpoint accepted an operation. Activation needs an observation from the chosen read path. Record the expected value, every observed logical value, the resolver or authoritative path used, and the observation time. Keep failed observations too.

No hand-waving here.

DMARC evaluation depends on an authenticated identifier being aligned with the visible From domain through SPF or DKIM, as RFC 7489 specifies. Publishing a DMARC TXT value therefore does not demonstrate that a message will pass DMARC. DNS readiness is the first gate. A later deliverability check should send through the configured path and retain machine-readable evidence from the resulting authentication evaluation; do not merge that outcome into the DNS publication result.

This separation pays off in an eval harness. Use fixtures for missing names, stale values, duplicate logical policies, a split TXT presentation, and a newer revision arriving during a retry. The score is deterministic: all expected logical values are observed for the current revision, or the domain remains blocked with a reason per record. Token-heavy explanations can be generated later for the UI, but structured evidence must drive the transition.

## Failure states the admin console must preserve

The most dangerous UI state is `pending` with no detail. It invites repeated clicks and hides whether the system is waiting for DNS visibility, seeing a conflicting value, or verifying an obsolete revision. Use a small state model such as `queued`, `writing`, `observing`, `ready`, and `blocked`, while retaining the record-level result beneath it.

Three failures deserve explicit handling. Partial progress means SPF may match while DKIM does not; retry the current revision and never activate early. Supersession means revision B has replaced revision A, so an A worker must be unable to mark the domain ready. An ownership collision means the target contains a different application-owned policy; surface the conflict instead of silently appending another policy string.

A compact evidence object is enough for the UI and audit trail:

```json
{
  "domain": "mail.customer.example",
  "revision": "sha256-of-canonical-intent",
  "state": "blocked",
  "records": [
    {
      "kind": "dkim",
      "owner": "saas1._domainkey.mail.customer.example",
      "matched": false,
      "observed": []
    }
  ]
}
```

Avoid storing only a prose error produced by a model. Prose changes, costs tokens, and is awkward to evaluate. Stable reason codes plus raw observations let the console explain the result without turning a prompt into the source of truth.

## Operational completion criteria

Before enabling mail for a tenant, confirm that the stored revision is current, every intended owner has an exact logical-value match, and the evidence came from the configured verification path after the latest write attempt. Then test the sending path separately and retain its authentication result. Activation should require both gates if the product promises a verified sending domain.

Run reconciliation on retry with the same desired state instead of inventing a repair workflow. Put bounded backoff and a deadline around observation, expose the last check time in the console, and alert on domains stuck beyond that deadline. Watch counts by reason code, not only a global success rate; a rise in `value_mismatch` tells a different story from a rise in read timeouts.

Finally, exercise the adapter contract in continuous integration with deterministic fixtures, then run a small integration suite against the real DNS path before deployment. The operational checklist is short in practice: revision fencing, record-level evidence, split-string normalization, collision protection, delayed activation, and a distinct end-to-end mail check. **The domain is ready when current evidence says it is, not when the job has merely stopped running.**

## Sources

- https://datatracker.ietf.org/doc/html/rfc7489
