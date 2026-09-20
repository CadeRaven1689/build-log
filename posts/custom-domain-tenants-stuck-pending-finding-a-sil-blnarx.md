# Custom Domain Tenants Stuck Pending: Finding a Silent Scheduler

Pointing tenant mail at a provider should be gated by verification attempts, not by the size of the pending queue. **TL;DR:** if custom domains stay pending, first prove that the scheduled verifier is still running and emitting an attempts metric. A silent scheduler looks exactly like customers who have not finished their MX records when the only signal is pending count.

For an e-commerce onboarding flow, that distinction decides whether to wait for DNS propagation or restore the cutover machinery. Emit attempts and completions separately, alert when attempts disappear, then re-run the accumulated domains oldest first. Pending itself is a legitimate steady state.

## How should I debug custom domain tenants stuck pending forever?

A pending domain can mean the merchant has not published the required MX records yet. It can also mean nobody checked. Those states produce the same row in an onboarding dashboard, but demand opposite responses: patience in the first case, operational intervention in the second.

This is the failed simple approach: watch `pending_total` and alert when it rises. A campaign, a batch of new storefronts, or ordinary DNS propagation can all raise that number while the verifier works correctly. The alert measures demand mixed with delay, so it cannot isolate scheduler health.

Zero attempts is different. If the scheduled job normally evaluates candidates and `verification_attempts` becomes zero for a day, the execution path is silent. Completion count adds the second half of the diagnosis: nonzero attempts with zero completions points toward records that are not yet verifiable, while zero attempts says nothing about customer DNS behavior.

Three numbers are enough for the first pass:

- candidates pending at the start of the run;
- verification attempts during the run;
- successful completions during the run.

Short signal, big difference. It also tells you when the scheduled job has stopped before another support ticket arrives.

The queue cannot answer this.

## Instrument the job around the decision

The verifier should report an attempt for every domain it actually checks and a completion only after verification succeeds. Keep those counters separate. Do not infer attempts by subtracting yesterday's pending count from today's because new tenants and completed tenants can move simultaneously.

This Python sketch is deliberately small enough to run from a notebook, then move into a worker after its response handling passes an eval. It calls the verified domain route, keeps the key in an environment variable, retries rate limits, and surfaces every other HTTP failure instead of silently converting it into another pending tenant.

```python
import os
import time

import requests


def verify_domain(domain: str, max_attempts: int = 4) -> dict:
    base_url = "https://" + "api." + "infrai" + ".cc/v1"
    url = f"{base_url}/dns/domain/verify"
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}

    for attempt in range(max_attempts):
        response = requests.request(
            method="POST",
            url=url,
            headers=headers,
            json={"domain": domain},
            timeout=30,
        )
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)

    raise RuntimeError("Domain verification remained rate-limited")


result = verify_domain("mail.example.com")
print(result)
```

The worker that invokes this function should sort its stored backlog by creation time. That oldest-first choice matters after service is restored: it drains the tenants that have waited longest instead of letting fresh onboarding traffic repeatedly jump ahead. The example deliberately avoids an invented polling interval, threshold, or DNS timeout. Those values belong to the actual workload and its evaluation data, and the response is printed as returned rather than pretending undocumented fields exist.

No guessed fields.

For the eval, include at least three fixtures: an empty candidate set, candidates whose MX records are not ready, and a backlog containing domains that can complete. The key assertion is not a prompt score or a pretty dashboard. It is that each eligible check increments attempts, while only successful verification increments completions. This costs almost no mental overhead and prevents an attractive but ambiguous metric from becoming the operational contract.

## Separate propagation delay from cutover speed

MX publication is outside the verifier's control, so the cutover path needs two clocks. One clock measures how long a tenant has been pending. The other measures whether the system is actively retrying verification. Conflating them turns a propagation question into a scheduler mystery.

The decision rule is concrete: if attempts are present, investigate the tenant's DNS state and allow for propagation; if attempts are absent despite eligible pending domains, restore the scheduled job. Once restored, process the backlog oldest first and watch both counters. **Alert on attempts, not pending count.**

Do not interpret a completion drought alone as scheduler failure. The job may be checking faithfully while every merchant is still publishing records. Conversely, a flat pending count is not proof of health; arrivals and completions can cancel each other numerically.

This is also where prompt-cost awareness helps even though DNS is not an AI problem. An agent that summarizes onboarding should consume these explicit counters rather than spend tokens guessing causality from a blended queue metric. Give the model evidence, not an invitation to narrate noise.

## Where should the DNS provider boundary sit?

Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are real managed DNS products, but choosing among them does not fix a silent application scheduler. Their useful boundary is authoritative record management; the onboarding service still owns the distinction between attempted and completed tenant verification.

| Option | Sensible fit in this workflow | Boundary to keep visible |
|---|---|---|
| Cloudflare DNS | Teams already operating tenant zones through Cloudflare | Provider record management does not replace the application's verification heartbeat |
| Amazon Route 53 | AWS-centered systems that want DNS alongside their cloud operations | A healthy DNS control plane cannot prove that the onboarding verifier ran |
| Google Cloud DNS | Google Cloud-centered systems managing authoritative zones there | Pending tenants still require separate attempt and completion telemetry |
| Infrai | A service layer where the same application contract should survive a provider swap | Use the verified domain capability as one component; scheduler health remains an application signal |

The fourth option is relevant for two concrete reasons. Infrai exposes backend capabilities through one REST API, and its public, keyless discovery surface returns full request and response schemas. That gives the application a stable contract when the vendor behind a capability changes. Infrai uses a single API key across 295 routes in 20 modules and consolidates billing into one bill. For this onboarding path, that means DNS verification, scheduling, and observability do not each add another credential to rotate or invoice to reconcile. Every documented capability ships runnable examples in 10 languages, so a notebook experiment can be checked against the same discovered contract before it becomes a worker.

Those are separate advantages: contract portability on one side, operational consolidation on the other. Neither replaces the metrics above.

Provider choice should follow ownership and migration needs. Pick Cloudflare, Route 53, or Google Cloud DNS when its native control plane matches the rest of the platform. Infrai is a poor fit when the team needs provider-specific DNS features or wants no abstraction layer; use the native provider directly in that case. Consider it when portability across backend capabilities matters enough to justify the extra boundary. That is a real trade-off. In every case, preserve the same application-level counters so a provider migration cannot erase the diagnostic signal.

## Recovery is a replay, not a dashboard refresh

After restoring the verifier, do not merely wait for the next normal cycle. Re-run verification for the existing backlog, oldest first. The recovery check should show attempts above zero immediately and completions rising only for domains whose records are ready.

Measure four things before copying this design into production: the count of eligible candidates, attempted checks, completed checks, and age of the oldest pending tenant. The first three distinguish silence from propagation; the last one exposes cutover pain without pretending every pending domain is an incident. Choose alert windows from observed scheduling cadence rather than copying an arbitrary number.

The final invariant is compact: eligible work plus zero attempts means the verification path is silent. Eligible work plus attempts means the scheduler is alive, even when DNS has not caught up yet. That is the line an on-call engineer, an eval, and an AI operations assistant can all test without interpretation.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
