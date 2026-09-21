# Agent Intent Guard

An async first-line guard for agentic workflows. It classifies an untrusted user
request against a developer-defined agent scope before the request reaches the
agent.

Rego policy generation is included as one example use case. The intent
classification logic itself is generic and can protect agents with other scopes.

## Classifications

- `valid_request`: The request matches the agent's purpose and capabilities and
  violates no boundary.
- `prompt_injection`: The request attempts to override, reveal, replace, or bypass
  the agent's instructions or safeguards.
- `out_of_scope`: The request is not prompt injection, but it does not belong to
  the configured agent scope or violates a boundary.

A valid classification is allowed only when it also meets the scope's configured
confidence threshold. Any low-confidence result is marked for review instead of
being acted on automatically.

## Project structure

- `main.py` contains the reusable Pydantic request, response, and decision models
  plus the async classification API.
- `examples/policy_requests.py` defines a Rego policy agent scope and demonstrates
  all three request classifications.

## Setup

This project requires Python 3.12 or later and uses
[uv](https://docs.astral.sh/uv/) for dependency management.

```bash
uv sync
```

Provide your TypeSafe API key through the environment:

```bash
export TYPESAFE_API_KEY="your-api-key"
```

The example also loads variables from a local `.env` file when one is present.

## Run the Rego policy example

```bash
uv run python -m examples.policy_requests
```

The example evaluates:

- A valid request to generate a Rego policy.
- A prompt-injection attempt.
- A benign request outside the Rego agent's scope.

For each request it prints the expected and actual classification, whether the
request is allowed, whether it requires review, and the model confidence and
probabilities.

## Use with another agent

Define an `AgentScope`, then pass it and the untrusted request to
`classify_intent`:

```python
import asyncio

from typesafe_sdk import AsyncTypeSafeClient

from main import AgentScope, IntentClassificationRequest, classify_intent


SUPPORT_AGENT_SCOPE = AgentScope(
    name="Customer support agent",
    purpose="Answer questions about customer accounts and product usage.",
    allowed_capabilities=(
        "Explain product features.",
        "Help troubleshoot account issues.",
    ),
    boundaries=(
        "Do not change account data.",
        "Do not reveal credentials or private customer information.",
    ),
    valid_examples=(
        "Help me reset my account preferences.",
    ),
    minimum_confidence=0.8,
)


async def run() -> None:
    async with AsyncTypeSafeClient() as client:
        request = IntentClassificationRequest(
            user_request="How do I change my notification settings?",
            agent_scope=SUPPORT_AGENT_SCOPE,
        )
        decision = await classify_intent(
            client,
            request=request,
        )

        if decision.allowed:
            print("Route to the support agent")
        elif decision.requires_review:
            print("Route to human review")
        else:
            print(f"Reject request: {decision.classification}")


asyncio.run(run())
```

The implementation follows the
[TypeSafe Python SDK async usage](https://docs.typesafe.ai/sdk/python/usage)
pattern: use `AsyncTypeSafeClient` as an async context manager and await
`system_one` through `classify_intent`.

`AgentScope` and `IntentClassificationRequest` reject missing, empty, or unknown
fields before an API call is made. The SDK response is parsed through a typed
`SystemOneResponse` subclass, and `IntentDecision` validates the final routing data.
