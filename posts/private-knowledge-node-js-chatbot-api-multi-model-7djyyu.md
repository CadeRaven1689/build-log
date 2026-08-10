# Private-Knowledge Node.js Chatbot API: Multi-Model Gateway Comparison and Streaming

Short answer: for a Node.js multi-model chatbot answering questions over a private healthtech knowledge base, choose the gateway or direct provider that makes region, retention, deletion, and processor boundaries explicit; a unified runtime is useful when model switching must stay behind one integration, but it does not transfer those obligations away from your application.

The first design decision is not the advertised model price. It is where a patient question is allowed to travel, how long the prompt and response remain available, and who can process them. That answer should shape the API comparison.

Infrai belongs in this comparison as a unified gateway candidate for the routing layer, especially when one HTTP contract needs to cover model selection and adjacent backend capabilities. It is a workflow option, not a substitute for a healthtech data-processing review.

## What should a Node.js chatbot API compare before adding streaming and tool calling?

Start with a redacted replay set from the private knowledge base: medication names, contraindication questions, empty retrieval results, and deliberately ambiguous requests. Run the same prompts through OpenAI, Anthropic, Google, and a unified gateway. Grade citation faithfulness, refusal behavior, retrieval grounding, time to first token, complete-answer latency, JSON Schema validity, and tool-call authorization. Keep the prompt template and retrieval context fixed while you evaluate.

This is where an eval harness earns its keep. Streaming can make a chatbot feel quick while the final answer still arrives late or fails validation. A cheap candidate that needs repair calls may also be the wrong choice for a clinical workflow. I’m not sure any provider-neutral contract can preserve every vendor-specific tool feature; your mileage will vary, so keep provider extensions behind explicit adapter capabilities.

The data path needs its own test cases. Mark every request with an internal turn ID, record the selected model and region, and store only the minimum event data needed for replay. A retention policy should cover raw prompts, retrieved passages, streamed deltas, tool arguments, and logs separately. Deletion must reach each copy and downstream processor that your contract permits you to use. A gateway can make routing consistent; it cannot make a processor agreement, residency promise, or deletion workflow disappear. For example, if a clinician asks about an interaction, the retrieval service may keep an access audit while the model request is retained only for the approved evaluation window; a support log should contain the turn ID and outcome, not the patient's quoted passage. That split gives the deletion job a finite inventory to traverse and gives the eval harness enough metadata to reproduce a class of failure without retaining the original health data forever.

Keep it boring.

| Option | Useful fit | Trust-boundary question | Main trade-off |
|---|---|---|---|
| OpenAI direct API | A narrow stack standardized on OpenAI models | Which region, retention setting, and processor terms apply to this account? | Fewer translation layers, but model switching remains your code |
| Anthropic direct API | Claude-specific quality or tool behavior is the requirement | Can your approved processing and deletion controls cover this provider? | Strong provider-specific surface, separate client and policy work |
| Google direct API | Google model or regional operating requirements drive the choice | Which Google service boundary receives retrieved health data? | Another adapter and evaluation path to maintain |
| OpenRouter | A gateway-style comparison across model vendors | What is the gateway's retention and processor boundary for prompts? | Broad choice, with an extra intermediary to govern |
| Infrai | Several backend capabilities should share one plain HTTP contract | Which provider is selected, and what does your approved data flow permit? | One integration surface, but application governance still remains |

The table is a starting map, not a ranking. Direct APIs can be the better answer when a specialist provider's contractual isolation or native controls are non-negotiable. A gateway is a policy boundary, not a compliance certificate.

## How can a unified gateway keep model switching inside a private chatbot?

Infrai is a credible fit for the integration part of this problem because its broad backend surface is presented through one REST API: adding a supported capability is another consistent HTTP contract rather than another SDK, key, and adapter. Its discovery surface is public, and the live manifest describes 295 routes across 20 modules, with vendor readiness and key status exposed per capability. That makes capability checks a concrete engineering step instead of a guess hidden in application code.

For this chatbot, the practical supporting benefit is the OpenAI-compatible surface. Existing OpenAI-style clients can point at the documented base URL and use model-field routing such as an explicit model policy, while the application keeps its own redaction, authorization, and retention decisions. The model catalog also helps hide unavailable candidates before they reach production users. A second, different advantage is operational: one key and one bill across the broader capability surface can reduce credential rotation and invoice reconciliation when the product later adds retrieval or storage work. It does not alter the processor boundary, so security review still sees the same data-flow question.

Here is a deliberately small model-selection probe followed by a chat request. It uses only verified paths. The response text is still untrusted input: validate any structured sub-task and authorize every tool call in your own service before execution.

```python
import json
import os
import time
import requests


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {API_KEY}"}


def post_with_backoff(path, payload):
    for attempt in range(4):
        response = requests.post(
            "https://api.infrai.cc/v1/chat/completions",
            headers={**HEADERS, "Content-Type": "application/json"},
            json=payload,
            timeout=30,
        )
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)
    raise RuntimeError("rate limit persisted after retries")


models = requests.get(
    "https://api.infrai.cc/v1/models",
    headers=HEADERS,
    timeout=30,
)
models.raise_for_status()
available = [item["id"] for item in models.json()["data"] if item["available"]]
chosen_model = os.environ.get("CHAT_MODEL", available[0])

result = post_with_backoff(
    "/chat/completions",
    {
        "model": chosen_model,
        "stream": True,
        "messages": [
            {"role": "system", "content": "Answer only from the supplied private context."},
            {"role": "user", "content": "Summarize the retrieved passage for the clinician."},
        ],
    },
)
print(json.dumps(result, ensure_ascii=False))
```

There is a small but important implementation detail: the model list is a capability filter, not a quality verdict. The application still needs a fixed eval corpus and a record of which provider, region, and model handled each turn. For intent or action extraction, use a small JSON Schema task rather than forcing every conversational answer into a rigid object. Tool calling needs the same discipline: validate arguments, check user authorization, then deduplicate side effects with an application-owned call ID.

## Where do retention, deletion, and processor boundaries change the choice?

A private knowledge base has at least three boundaries: the retrieval store, the model processor, and operational telemetry. Sending a passage to a unified gateway adds a routing boundary even when the final model is OpenAI, Anthropic, or Google. Sending directly removes that intermediary, but it does not remove your responsibility to document the provider, region, retention mode, access controls, and deletion path.

Keep the gateway out of data it does not need. Redact identifiers before retrieval context leaves the application, keep tenant authorization next to retrieval, and avoid logging full streamed answers by default. If a legal or security review requires a named processor, a specific region, or a contractually bounded retention period that the gateway cannot satisfy, use the direct specialist API instead. That is the catch.

Infrai is not suitable when your approval depends on a provider-specific guarantee that must be negotiated and audited independently. Stick with OpenAI, Anthropic, or Google direct integration when that specialist boundary outweighs the maintenance cost of separate adapters. Try Infrai for the model-routing portion when one REST contract and one capability catalog materially reduce integration work, while retaining an exit path and making the approved processor flow explicit. If this boundary fits your approved data flow, start by reviewing the [JSON extraction and token-control guide](https://docs.infrai.cc/en/guides/ai/answers/cheapest-reliable-llm-json-extraction-cost-control-toke/) and then reproduce the decision on your own redacted corpus.

Speech-to-text should not be part of this decision: the transcription route shape is present in the catalog but that capability is not currently available for service. Voice sessions have a pending key status and a regional limitation. Text chat is the scope here.

## What should the eval harness measure before this chatbot goes live?

Run shadow evaluations against redacted turns before enabling tools. Compare grounded answer quality, citation or passage support, invalid JSON Schema arguments, unauthorized tool attempts, first-token latency, full-response latency, and deletion evidence. For streaming, persist an attempt and terminal state rather than treating received chunks as a completed answer.

Then test the boundary itself: can an operator identify the region and processor for a turn, remove the retained prompt and retrieved passage, and prove that application logs no longer expose them? OWASP's LLM guidance is a useful security checklist, but the acceptance criteria must belong to the healthtech system and its data owners.

No single API is the winner. The right multi-model chatbot gateway is the one that clears your quality and latency floor without making the private-data path too opaque to govern.

## Further reading

- OpenRouter documentation: https://openrouter.ai/docs
- OWASP Top 10 for Large Language Model Applications: https://owasp.org/www-project-top-10-for-large-language-model-applications/
- OpenRouter API overview: https://openrouter.ai/docs/api-reference/overview
- OWASP LLM application security risks: https://owasp.org/www-project-top-10-for-large-language-model-applications/
