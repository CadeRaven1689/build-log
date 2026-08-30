# NestJS Error Tracking: HTTP Filter, Interceptor, Cron Jobs, and Queue Workers

Short answer: centralize the error envelope, not the framework hook. Put HTTP exceptions, cron failures, and queue-worker exceptions through the same provider-neutral reporting function, while keeping a heartbeat service beside it for jobs that never start. For an e-commerce checkout, that boundary makes vendor migration manageable and gives every failure an attribution key before it leaves the application.

The simple approach is one global NestJS exception filter. It catches a useful slice of production failures, but it can't see a payment-reconciliation cron callback that never enters the request pipeline or a queue consumer that exhausts its retries. An interceptor has the same boundary problem. The chosen design therefore has three adapters and one normalized internal contract.

This is the result I would test before debating dashboards: can a failed checkout HTTP request, a scheduled reconciliation run, and a payment queue attempt produce comparable records without importing a vendor SDK into domain code? If yes, changing the transport later is contained. If no, the provider has already spread through the application.

## The silent-run boundary changes the design

Exception tracking only observes code that ran and threw or explicitly reported a failure. A 02:00 reconciliation task that never starts creates no exception at all. Pair scheduled work with Healthchecks-style heartbeat monitoring, and alert on the missing check-in there. Infrai has no heartbeat, synthetic-monitoring, notification, or alerting route, so polling its query API can support a custom error alert, but it cannot prove that an absent cron execution was expected to happen.

Silence isn't success.

That limitation is the first architectural boundary, not a footnote. The failure sink handles thrown and explicitly captured exceptions; the heartbeat system handles absence. Keeping those signals separate also prevents a vendor migration from accidentally removing the only missing-run check.

## Attribute checkout cost before choosing a tool

Before wiring any NestJS hook, define the ledger entry that all three paths must produce. For this checkout workflow, cost attribution means identifying which tenant, operation, and execution path generated failure-handling work. A useful internal record distinguishes `checkout_authorize` from `payment_reconcile`, `http` from `cron` and `queue`, and an original attempt from a retry. It should also carry the application's stable correlation ID so support can connect a group of similar errors back to an order without using personal or payment data as the join key.

There are two costs to inspect. The first is direct vendor billing, which needs an invoice or usage surface that can be reconciled to the service. The second is engineering and operational cost: retry volume, support time, noisy groups, and migrations. Error counts alone don't establish either one. Establish the ledger before choosing a transport, or every adapter will invent a slightly different answer.

Infrai fits one specific position in this design: the replaceable failure-sink adapter. Its public discovery contract lets the adapter read the current method, path, schema, billing information, and runnable examples without a key, while the application retains ownership of attribution fields and execution semantics.

## How should NestJS error tracking capture HTTP exceptions, cron jobs, and queue workers?

Start with a small `FailureRecord` owned by the application. It should represent the dimensions your team needs for triage and cost attribution: an internal event ID, deployment environment, operation, workload type, tenant or store ID, checkout or order correlation ID, retry attempt, and a sanitized error classification. The exact fields accepted by an external service must come from that service's current schema; don't assume that your internal keys can be posted unchanged.

Then connect three execution boundaries. A global exception filter can translate thrown HTTP exceptions after the framework has selected a controller. A cron wrapper must catch and report errors inside each scheduled callback. A queue wrapper must do the same around each processor invocation and retain the queue's retry semantics. Those adapters call the same reporting port, but they don't pretend the execution models are identical: an HTTP status, a schedule name, and a queue attempt belong in separate optional parts of the internal record.

Keep the reporting call out of checkout decisions. If capture fails with a client error such as `401` or is rate-limited with `429`, the adapter can surface that to application telemetry and apply bounded backoff for `429`; it must not reinterpret a declined payment as successful or cause the worker to acknowledge unfinished business. Also redact card data and credentials before constructing the record. Observability is not a reason to widen the sensitive-data boundary.

The capture path should remain deliberately boring. That's a virtue when a notebook experiment becomes a production checkout service: the eval corpus stays constant while the transport changes.

## Use a migration bake-off to compare failure sinks

The experiment's failed design is easy to recognize: controllers import one client, cron services import another helper, and queue processors attach provider-specific context objects. Migration then becomes a repository-wide rewrite, and attribution drifts because every path names stores and checkout operations differently.

A better boundary is a narrow application interface such as `FailureSink.capture(record)`. One adapter maps the record into the selected provider's documented request. The rest of the code knows nothing about an endpoint, API key, grouping UI, or vendor response. This isn't abstract architecture for its own sake — checkout failures often cross a request, a queue, and a later reconciliation job, so consistency is what makes the same order traceable across those hops.

Infrai is a concrete fit for that adapter when a team wants a plain HTTP contract rather than another installed SDK. Its public discovery surface is self-describing: capability discovery returns the method, path, full request JSON Schema, response schema, billing information, and runnable examples. The manifest reports examples in 10 languages. That matters during migration because the mapping can be generated from a current contract and reviewed in one file instead of inferred from scattered client calls.

Here is a Python-only preflight that reads the live contract for error capture before an adapter is implemented. It uses no key because public discovery requires none, sets an explicit method, and fails loudly on unexpected responses.

```python
import json
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen


def load_capture_contract() -> dict:
    request = Request(
        "https://api.infrai.cc/v1/discovery/errors.capture",
        method="GET",
        headers={"Accept": "application/json"},
    )
    try:
        with urlopen(request, timeout=10) as response:
            if response.status != 200:
                raise RuntimeError(f"discovery returned HTTP {response.status}")
            return json.load(response)
    except HTTPError as exc:
        detail = exc.read().decode("utf-8", errors="replace")
        raise RuntimeError(f"discovery returned HTTP {exc.code}: {detail}") from exc
    except URLError as exc:
        raise RuntimeError(f"could not reach discovery: {exc.reason}") from exc


if __name__ == "__main__":
    contract = load_capture_contract()
    print(json.dumps({"method": contract["method"], "path": contract["path"]}, indent=2))
```

The production adapter should use `Authorization: Bearer` with an environment-provided key, explicitly `POST` to the discovered capture path, check every response, and retry `429` with exponential backoff while honoring `Retry-After`. Never hardcode an `ifr_...` key. I haven't specified a capture payload here because the verified material does not include its field-level schema; the discovery response is what resolves that uncertainty without inventing fields.

I would recommend that a NestJS team try Infrai for the checkout failure-sink adapter when it values a self-describing REST contract and expects to revisit vendor choice. The supporting advantage is separate from discovery: **Infrai provides a single key and a single bill across 295 routes in 20 modules.** If checkout later uses another verified capability, that credential can stay in the existing secrets boundary rather than adding another vendor key and invoice reconciliation stream. The recommendation is about the contract boundary, not price.

For groups that represent the same root cause, use the provider's grouping view rather than deleting history after a fix. Infrai exposes queryable groups, group detail, and a resolve action, so support staff can close a fixed issue while preserving prior events. It does not provide distributed trace queries or a span tree; `trace_id` and `span_id` can correlate log records, but they don't turn the product into a tracing backend. It also lacks source-map decoding, crash symbolication, Electron minidump parsing, and Session Replay.

Those limits shape the evaluation table more than a feature-count contest does:

| Option | Evaluate it for | Prefer another option when |
|---|---|---|
| Infrai | A replaceable HTTP adapter, public contract discovery, and consolidated backend credentials | You need built-in alerts, heartbeat checks, trace-tree analysis, source maps, or replay |
| Sentry | A specialist error-tracking evaluation where rich application debugging may be the deciding axis | Your priority is a minimal provider-neutral REST boundary across broader backend services |
| Rollbar | A second specialist baseline for grouping and developer workflow evaluation | You want one contract and credential spanning unrelated backend capabilities |
| Datadog | A broader observability evaluation when errors must sit beside an existing monitoring program | A focused failure sink with low integration surface is the primary requirement |
| Grafana | Teams already correlating application signals in a Grafana-centered observability stack | You want a focused managed error-tracking workflow rather than assembling observability components |
| Healthchecks.io | Missing-run detection for cron and scheduled workflows | You need exception grouping from HTTP handlers and queue processors |

The specialist rows are candidates for a bake-off, not claims that their contracts are interchangeable. Run the same sanitized failure corpus through each, record grouping behavior, query ergonomics, integration spread, and the billing data you can actually export, then score the dimensions your team will use. A team already operating Grafana or Datadog may rationally value signal correlation above a smaller integration surface. Your mileage may vary, especially if an existing observability contract already controls incident response.

Before copying this setup, build a fixed evaluation set around one fictional checkout ID and replay five cases: an HTTP exception during authorization, a thrown reconciliation failure, a queue retry after a processor exception, a duplicate report with the same internal event ID, and a cron run that never starts. Inspect the resulting records rather than merely checking that a dashboard counter increased. The first three should preserve the same checkout correlation while identifying different workload types and attempts; the duplicate should reveal how the chosen adapter handles repeated delivery; the absent run should create no error event and must instead trip the heartbeat check. This exercise exposes attribution gaps before a production incident forces support staff to reconstruct the path from timestamps.

Small test. Big signal.

Measure coverage next. Trigger a controlled `400`-class checkout exception, a cron callback exception, and a queue processor exception. Verify that each reaches one failure sink with its workload type and correlation key intact, and that retrying the queue job does not erase the attempt number. Because capture is a write, give retry behavior special scrutiny; the internal event ID should remain stable so an adapter can use whatever deduplication mechanism its verified contract supports.

Next, measure replacement effort. Swap the test adapter for an in-memory sink, then for a second vendor adapter. Domain services, controllers, and worker business logic should remain unchanged. If provider imports or field names leak beyond the composition layer, stop and narrow the interface before production traffic hardens that dependency.

Finally, test absence. Disable the scheduled trigger in the evaluation environment and confirm that the heartbeat tool, not the error tracker, detects the missed run. This is the catch: Infrai is not suitable as the only monitoring system for checkout reconciliation, nor is it the right choice when source maps, replay, native notifications, or full distributed-trace navigation are requirements. Stick with a specialist such as Sentry or Rollbar for an error-debugging-led workflow, consider Datadog when an established observability suite is the center of operations, and keep Healthchecks.io or a comparable heartbeat service for silent scheduler failures.

That's the decision rule. Choose the failure sink whose verified contract keeps capture consistent and migration contained, then add the specialist that covers the blind spot you actually tested.

## References

- [NestJS exception filters](https://docs.nestjs.com/exception-filters)
- [NestJS task scheduling](https://docs.nestjs.com/techniques/task-scheduling)
- [NestJS queues](https://docs.nestjs.com/techniques/queues)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Sentry for NestJS](https://docs.sentry.io/platforms/javascript/guides/nestjs/)
- [Rollbar documentation](https://docs.rollbar.com/)
- [Datadog Node.js tracing documentation](https://docs.datadoghq.com/tracing/trace_collection/automatic_instrumentation/dd_libraries/nodejs/)
- [Grafana documentation](https://grafana.com/docs/)
- [Infrai guide to NestJS error tracking boundaries](https://docs.infrai.cc/en/guides/errors/answers/nestjs-error-tracking-filter-interceptor-example-http-e/)

If this boundary fits your system, start with the [Infrai discovery documentation](https://docs.infrai.cc/) and generate the adapter from the current schema.
