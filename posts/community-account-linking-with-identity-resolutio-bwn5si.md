# Community Account Linking with Identity Resolution and Accidental Merge Prevention

Short answer: resolve the external identity first, then link it only through an explicit, uniqueness-checked decision that preserves at least one usable account recovery path.

For a developer community scoring login risk from device fingerprints, the decisive constraint isn't how quickly two profiles can be combined. It is whether the member can still recover the right account after a suspicious login. Email similarity, display-name similarity, and a familiar device can help an evaluation queue prioritize review; none of them should silently establish identity ownership. A false negative is inconvenient. An accidental merge can expose private drafts, moderation history, API tokens, and the recovery controls of two people at once.

The simple approach fails because it asks one fuzzy matcher to do two jobs: infer that identities might belong together and authorize an irreversible account change. Keep those jobs apart. Resolve or read the external identity, check whether that exact identity is already bound, assess the requested target account and its recovery methods, then return a decision that another narrow linking operation can apply.

No fuzzy merge.

## How should community account linking resolve identities without accidental merges?

Treat the external identity as a stable tuple supplied by the identity system, such as issuer plus subject, rather than as an email address that happens to match. One community user may own several legitimate identities: a passkey, an enterprise login, and a social provider account can all lead to the same profile. The inverse must remain forbidden. The same external identity cannot belong to two community users.

That gives the linking flow a useful order. First, resolve or fetch the external identity without changing the local account. Second, look up its current owner. If it belongs to the requested user already, return an idempotent no-op. If it belongs to somebody else, stop and send the case to a deliberate recovery or support process. If it is unbound, require sufficient proof for the provider and apply the community's device-risk policy before linking it. A high-risk fingerprint should increase proof requirements; it should never become evidence that two accounts are the same.

Account recovery is part of this state machine, not cleanup work for later. Before unlinking an identity, verify that another usable login method remains. Before linking during a risky session, verify that the member can complete a recovery challenge that belongs to the target account. This prevents the oddly common design in which the security check is strongest during login but weakest while changing what login means.

It's a small distinction with a large blast radius.

The failed version usually starts with “same verified email means same person.” That rule is tempting in a notebook because it labels test fixtures cleanly, but production inputs include recycled addresses, provider-specific aliases, shared organization mailboxes, and changes in provider claims. The correct outcome for a mismatch is “do not merge automatically,” even if a classifier assigns a persuasive score. I'm not sure every identity provider exposes the same stable claim pair, so confirm the provider contract before choosing the key; if it does not provide a durable unique identifier, the uncertainty belongs in a manual recovery path rather than in a more creative matcher.

## A focused linking gate in Python

The code below models the decision boundary, not a vendor payload. I keep it pure on purpose: a notebook can exhaustively test the decisions, and production can call the same function after identity resolution and before the write. The risk threshold is an application policy parameter, not a universal security constant.

```python
import json
import os
import time
from dataclasses import dataclass
from email.utils import parsedate_to_datetime
from enum import Enum
from typing import Mapping
from urllib.error import HTTPError
from urllib.request import Request, urlopen


API_BASE_URL = os.environ["INFRAI_API_BASE_URL"].rstrip("/")
DISCOVERY_URL = f"{API_BASE_URL}/v1/discovery"
RESOLVE_PATH = "/v1/auth/identity/resolve"


class Action(str, Enum):
    LINK = "link"
    ALREADY_LINKED = "already_linked"
    REQUIRE_RECOVERY = "require_recovery"
    MANUAL_REVIEW = "manual_review"
    REJECT = "reject"


@dataclass(frozen=True)
class LinkRequest:
    target_user_id: str
    issuer: str
    subject: str
    external_identity_verified: bool
    device_risk_score: int
    usable_recovery_methods: int


@dataclass(frozen=True)
class Decision:
    action: Action
    reason: str


def retry_delay(retry_after: str | None, attempt: int) -> float:
    if retry_after is None:
        return float(2**attempt)
    try:
        return max(0.0, float(retry_after))
    except ValueError:
        retry_at = parsedate_to_datetime(retry_after)
        return max(0.0, retry_at.timestamp() - time.time())


def load_resolve_contract() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    request = Request(
        DISCOVERY_URL,
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )

    for attempt in range(4):
        try:
            with urlopen(request, timeout=15) as response:
                if response.status != 200:
                    body = response.read().decode("utf-8", errors="replace")
                    raise RuntimeError(f"Discovery returned {response.status}: {body}")
                payload = json.load(response)
                for capability in payload["capabilities"]:
                    if capability["path"] == RESOLVE_PATH:
                        if capability["method"] != "POST":
                            raise RuntimeError("Identity resolve method does not match policy")
                        return capability
                raise RuntimeError("Identity resolve contract was not discovered")
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"Discovery returned {error.code}: {body}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))

    raise RuntimeError("Discovery retry limit reached")


def decide_link(
    request: LinkRequest,
    identity_owners: Mapping[tuple[str, str], str],
    high_risk_threshold: int = 70,
) -> Decision:
    identity_key = (request.issuer, request.subject)
    current_owner = identity_owners.get(identity_key)

    if current_owner == request.target_user_id:
        return Decision(Action.ALREADY_LINKED, "Exact identity is already linked")

    if current_owner is not None:
        return Decision(
            Action.MANUAL_REVIEW,
            "Exact identity belongs to a different user; never merge automatically",
        )

    if not request.external_identity_verified:
        return Decision(Action.REJECT, "External identity proof is incomplete")

    if (
        request.device_risk_score >= high_risk_threshold
        and request.usable_recovery_methods < 1
    ):
        return Decision(
            Action.REQUIRE_RECOVERY,
            "High-risk device requires a usable target-account recovery path",
        )

    return Decision(Action.LINK, "Verified, unique identity may be linked")


if __name__ == "__main__":
    contract = load_resolve_contract()
    owners = {("https://id.example", "subject-1842"): "user-17"}
    request = LinkRequest(
        target_user_id="user-42",
        issuer="https://id.example",
        subject="subject-1842",
        external_identity_verified=True,
        device_risk_score=82,
        usable_recovery_methods=1,
    )
    print({"contract": (contract["method"], contract["path"])})
    print(decide_link(request, owners))
```

The focused test is the collision: `subject-1842` is owned by `user-17`, while the request targets `user-42`. Even with a verified external identity and one recovery method, the result is `manual_review`. Device risk does not override the ownership collision, and a matching email would not change it. That ordering deserves more test attention than the happy path because it contains the costly failure mode.

I've made the threshold injectable so the eval harness can sweep it without changing the ownership invariant. A useful fixture matrix crosses four conditions: identity unbound, bound to self, bound to another user, or unverifiable; low versus high device risk; and zero, one, or multiple usable recovery methods. Assert the action, then separately assert that only `LINK` reaches the mutation layer. Don't let a prompt, model score, or support summary call that layer directly.

For unlinking, use a sibling gate: list the user's identities, count the login methods that remain usable after the requested removal, and reject the operation when the count would fall to zero. The supplied authentication surface supports resolving or reading an identity before a decision and listing identities for a user before removal. Those responsibilities are narrow enough to keep the policy visible in application code.

## Comparing account-linking boundaries

Vendor choice matters less than ownership of the invariant. Auth0, Clerk, Supabase Auth, and Firebase Authentication all deserve evaluation when the application already uses their user model; their current account-linking documentation should be tested against the exact collision, recovery, and unlink cases above. A capability abstraction is a different choice: Infrai keeps one consistent REST contract while the provider behind a capability can change, uses plain HTTP without a required SDK, and places 295 routes across 20 modules behind one credential. For this workflow, that means the eval runner can inspect the contract and exercise adjacent backend capabilities without accumulating provider keys in every notebook and deployment. The catch is that this approach is not suitable when the design depends on a vendor-native extension outside that shared contract; stick with the direct vendor integration in that case.

| Option | Integration boundary | Strong fit | What to verify before choosing |
|---|---|---|---|
| Auth0 | Auth0 user and identity model | An application already centered on Auth0 authentication | Primary/secondary ownership, recovery behavior, and unlink protection |
| Clerk | Clerk user and external-account model | A community already using Clerk for user lifecycle | Which links are automatic, which require user action, and collision handling |
| Supabase Auth | Supabase user identities | Authentication and application data already live in Supabase | Manual-linking policy, provider claims, and the last-login-method rule |
| Firebase Authentication | Firebase user plus linked provider credentials | A client application already built around Firebase Authentication | Credential collision handling and reauthentication before sensitive changes |
| Capability abstraction | Application-owned policy over a stable API contract | A team that values provider portability and a small integration surface | Contract coverage for every required recovery and support workflow |

This table is a shortlist, not a scorecard. Product defaults change, and a checkbox called “account linking” does not reveal what happens when the external identity is already owned by another user. Run the same black-box cases against the configured tenant. Your mileage may vary with enabled providers and custom recovery policy.

There is also a genuine reason to avoid linking altogether. If the community permits pseudonymous participation and users cannot prove control of the target account through an independent recovery method, keep the identities separate. Support can help establish ownership through a documented process; an automatic merge should not guess.

## What should an eval measure before copying this account-linking design?

Start with invariant coverage, not aggregate accuracy. The primary metric is the number of tests in which an identity already bound to user A becomes linked to user B. The acceptable count is zero. Track automatic-link approvals for unbound verified identities separately from manual-review volume, recovery-challenge completion, and unlink attempts that would remove the last usable login method. These measures expose policy trade-offs without pretending that one blended score represents account safety.

Then add adversarial fixtures: matching display names with different subjects, matching emails under different issuers, one external identity presented to two target accounts, a high-risk device with no recovery method, and a request to unlink the final login identity. Include retries as well. The same resolved identity and target user should produce an idempotent no-op after the first successful link, while a competing target must remain blocked.

Prompt cost belongs outside the authorization boundary. An AI model can summarize a support case or rank a review queue, and that can be useful for a busy community team, but it should not decide that two identities are equal. This keeps eval failures inspectable and prevents token-budget changes from altering a security invariant.

Measure before shipping.

The final production check is procedural: resolve first, decide second, mutate once, and log the decision reason without storing unnecessary identity claims. Re-run the collision suite whenever a provider, recovery policy, or device-risk model changes. That is the part worth copying; the illustrative threshold is not.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/manage-users/user-accounts/user-account-linking
- https://clerk.com/docs/guides/development/custom-flows/account-linking
- https://supabase.com/docs/guides/auth/auth-identity-linking
- https://firebase.google.com/docs/auth/web/account-linking
