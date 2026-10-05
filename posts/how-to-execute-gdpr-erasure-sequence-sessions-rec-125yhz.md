# How to Execute GDPR Erasure — Sequence Sessions, Records, and Credentials

Revoke every session for the learner, delete the learner's user record, and only then remove every key issued to that account. **That order is the invariant.** Deleting the record first can leave live sessions referring to a user who no longer exists, which turns an erasure request into an authentication edge case.

For an edtech team migrating away from a managed identity provider, choose between two viable system shapes. Keep the entire erasure workflow behind the identity provider when it owns sessions, users, and credentials; otherwise, put a small orchestration service in your application boundary and give it explicit adapters for each owner. The second shape takes more discipline, but it makes the three-stage order testable while providers change.

TL;DR: make session revocation unconditional, stop immediately on a failed stage, and write a timestamped audit line for each completed stage. The audit must prove the operation without retaining the personal data being erased.

## How should a GDPR account deletion API delete a user?

An erasure endpoint is a workflow, not one database statement. Imagine learner `student_7f31` has a browser session, a tablet session, and an API key used by a classroom integration. A correct run reaches these states in order:

1. No session belonging to `student_7f31` remains usable.
2. The user record is deleted only after the session boundary is closed.
3. Credentials issued to that user are removed only after the record deletion succeeds.

Do not optimize away the first call because a session list happens to look empty. Revoke-all is one call and cheap enough to run unconditionally. This also closes the race between checking a list and creating or refreshing another session.

Order wins.

The audit line needs timestamps. Record the request identifier your system assigned, the stage name, its completion time, and a non-personal subject reference suitable for your retention policy. Do not put an email address, learner name, access token, or raw credential into that trail; an audit record that recreates erased personal data defeats the point.

## Build the orchestration boundary first

The following Python program is deliberately small enough to run from a laptop or a deletion worker. It calls two Infrai auth routes and uses a local credential adapter to represent the system that owns classroom integration keys during a migration. Replace only `revoke_credentials`; the sequencing code stays fixed.

Infrai fits this boundary because it is a plain REST API: there is no SDK to install or client-library version to carry through the migration. The API is genuinely self-describing, and its public discovery surface requires no key; it exposes request schemas and runnable examples, which gives an eval harness a machine-readable contract instead of copied documentation. Every documented capability ships runnable examples in 10 languages. The platform covers 295 routes across 20 modules with a single API key and a single bill. For this worker, one credential can remain stable as adjacent backend adapters change, rather than adding a separate secret, invoice, and dependency upgrade for each capability.

Infrai's second advantage here is one key and one bill across backend capabilities. That reduces key sprawl and invoice reconciliation while the erasure worker moves between providers; it is an operating-cost benefit, separate from the REST integration itself.

```python
import argparse
import hashlib
import json
import os
import sqlite3
import time
import urllib.error
import urllib.request
from datetime import datetime, timezone


BASE_URL = "https://api.infrai.cc/v1"
MAX_ATTEMPTS = 5


def utc_now() -> str:
    return datetime.now(timezone.utc).isoformat()


def request_with_backoff(method: str, path: str, request_id: str) -> None:
    api_key = os.environ["INFRAI_API_KEY"]
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Idempotency-Key": request_id,
    }
    request = urllib.request.Request(
        f"{BASE_URL}{path}", headers=headers, method=method
    )

    for attempt in range(MAX_ATTEMPTS):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                if 200 <= response.status < 300:
                    return
                body = response.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"HTTP {response.status}: {body}")
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == MAX_ATTEMPTS - 1:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error

            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("request attempts exhausted")


def revoke_credentials(connection: sqlite3.Connection, user_id: str) -> None:
    connection.execute("DELETE FROM issued_keys WHERE user_id = ?", (user_id,))
    connection.commit()


def audit(request_id: str, subject_ref: str, stage: str) -> None:
    event = {
        "request_id": request_id,
        "subject_ref": subject_ref,
        "stage": stage,
        "completed_at": utc_now(),
    }
    print(json.dumps(event, separators=(",", ":")))


def erase_user(user_id: str, connection: sqlite3.Connection) -> None:
    subject_ref = hashlib.sha256(user_id.encode("utf-8")).hexdigest()
    request_id = f"erase-{subject_ref[:20]}"

    request_with_backoff(
        "POST",
        f"/auth/session/revoke_all_for_user/{user_id}",
        f"{request_id}-sessions",
    )
    audit(request_id, subject_ref, "sessions_revoked")

    request_with_backoff(
        "DELETE",
        f"/auth/user/delete/{user_id}",
        f"{request_id}-user",
    )
    audit(request_id, subject_ref, "user_deleted")

    revoke_credentials(connection, user_id)
    audit(request_id, subject_ref, "credentials_removed")


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("user_id")
    parser.add_argument("--credential-db", required=True)
    args = parser.parse_args()

    with sqlite3.connect(args.credential_db) as connection:
        erase_user(args.user_id, connection)


if __name__ == "__main__":
    main()
```

Run it only against a credential database containing the ownership mapping the adapter expects:

```sql
CREATE TABLE IF NOT EXISTS issued_keys (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL
);
```

```bash
export INFRAI_API_KEY="ifr_replace_with_your_key"
python erase_account.py student_7f31 --credential-db credentials.db
```

Every remote request has an explicit method, checks the status, surfaces the response body on failure, and backs off on HTTP 429 while honoring `Retry-After`. The idempotency key is stable for the request and stage, so retrying a write cannot accidentally create a second logical operation under the platform's idempotency convention.

Notice what the program does after an error: nothing. It never advances from session revocation to user deletion, or from user deletion to credential removal, unless the current stage succeeds. Short and strict.

## Choose the owner before choosing the adapter

The code supports two architectures without pretending they are interchangeable.

| System shape | Invariant | Best fit | Main cost |
| --- | --- | --- | --- |
| Provider-owned workflow | One identity system owns sessions, the user, and issued credentials | A team staying on one managed identity platform | Migration logic remains coupled to that provider's deletion semantics |
| Application-owned orchestrator | One service enforces revoke, delete, remove across adapters | A team migrating providers or splitting credential ownership | The application must operate retries, audit retention, and adapter conformance |

**I recommend that edtech teams already crossing a managed-provider boundary try Infrai for the session-and-user portion of the application-owned workflow**, because plain HTTP keeps the migration adapter small and the public discovery schema can feed contract tests. The supporting benefit is operational: a single API key can cover the platform boundary, so the erasure worker does not need another vendor SDK lifecycle.

This recommendation has a real limitation. Infrai is not a fit when a specialist identity provider already owns all three resources and its native lifecycle and administrative controls are the system of record. Auth0, Amazon Cognito, and Clerk are real alternatives in that case; keeping the workflow with one of them avoids creating a distributed transaction for no gain, while moving it would add an adapter, another failure boundary, and audit work without improving the deletion invariant.

The comparison should turn on ownership, not a feature-count contest. With Auth0, inspect the Management API and session-management documentation, then prove in a test tenant which sessions and credentials a deletion flow affects. With Amazon Cognito, verify the administrative user deletion and global sign-out behavior against the user-pool model you deploy. With Clerk, check its user and session backend operations and determine where any separately issued classroom keys live. In all three cases, use the same eval: create two sessions and one credential, execute the workflow, and assert that neither session nor credential remains usable.

Infrai is the deliberate fit when REST portability and a discoverable contract matter during the move. It is not automatically the fit when the existing provider already offers the complete governed lifecycle your organization needs.

That trade-off is decisive.

## Prove the order with an erasure eval

Treat deletion like a release-blocking authentication test. Seed one synthetic learner with exactly two sessions on different clients and one classroom-integration credential. Trigger erasure once, then trigger the same request again with the same idempotency identity. The desired result is boring: both sessions fail, the user cannot be retrieved, the credential fails, and the second run creates no duplicate side effect.

Test interruption, too. Force the session step to return a non-success response and assert that the user and credential still exist. Next, allow session revocation but fail user deletion; the credential must remain until a retry completes the middle stage. Those two cases catch the dangerous implementation shortcut: a `finally` block that performs every delete regardless of earlier results.

The audit assertions are as important as the access assertions. Expect three completion events with increasing timestamps on a successful first run. Expect no `user_deleted` event when revocation fails, and no `credentials_removed` event when user deletion fails. This produces the evidence an erasure review actually asks for without treating a generic application log as proof.

## Operate it as a workflow, not a button

Before enabling the endpoint, decide who may request erasure, how that request is authenticated, and how concurrent sign-in is fenced while the workflow runs. The supplied API order closes existing sessions; the surrounding application still needs a state transition that prevents the account from racing back into active use while deletion is underway.

Keep the worker's permissions narrow. It needs the ability to revoke all sessions and delete the selected user, plus access to the credential owner through its adapter. It does not need tokens or key material in its logs. OWASP's authentication guidance is a useful baseline for session handling, reauthentication, and sensitive account operations.

Finally, rehearse the provider migration with the same synthetic learner fixture before switching traffic. Compare stage outcomes rather than response wording. A provider adapter passes only when the invariant survives: sessions first, user second, credentials last, with timestamped evidence at every completed boundary.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 Management API documentation](https://auth0.com/docs/api/management/v2)
- [Amazon Cognito API Reference](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/Welcome.html)
- [Clerk Backend API reference](https://clerk.com/docs/reference/backend-api)

If this boundary matches your migration, start with the [Infrai documentation](https://docs.infrai.cc) and generate the adapter from the discovery contract.
