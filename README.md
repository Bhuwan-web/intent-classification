# Agent Intent Classification

A small, typed classifier that compares an untrusted request with a declared
agent scope before the request is handed to the agent.

## Visual introduction

[View the visual introduction](https://bhuwan-web.github.io/intent-classification/).

## Problem

An agent should receive requests that match its purpose. Application code,
however, needs a predictable value to distinguish a matching request from an
unrelated request or an attempt to override the agent's instructions.

This project supplies that value. It does not execute the downstream agent,
reject a request, or send a request to human review. Those actions belong to the
application that consumes the classification.

## Solution

The caller provides two things:

1. The untrusted user request.
2. An `AgentScope` describing the agent's purpose, capabilities, boundaries,
   valid examples, and confidence threshold.

The classifier sends the request and the descriptive parts of the scope to the
TypeSafe `system_one` API. It returns one of exactly three classifications:

- `valid_request`: the request matches the purpose and capabilities and violates
  no boundary.
- `out_of_scope`: the request is not prompt injection, but falls outside the
  scope or violates a boundary.
- `prompt_injection`: the request tries to override, reveal, replace, or bypass
  the agent's instructions or safeguards.

```text
user request + agent scope
            |
            v
    classify_intent(...)
            |
            v
 valid_request | out_of_scope | prompt_injection
```

The result also contains the selected label's confidence, the probability map,
and the configured minimum confidence.

### Classification versus computed flags

`classification` is the canonical model result. `allowed` and
`requires_review` are computed convenience fields; they are not additional
classifications or evidence that the application routed the request anywhere.

```text
allowed =
    classification == valid_request
    AND confidence >= minimum_confidence

requires_review =
    confidence < minimum_confidence
```

There is no `reject` field. A caller may choose what to do with an
`out_of_scope` or `prompt_injection` result, but that behavior is outside this
repository.

### One concrete request

The example declares a Rego policy agent and submits:

```text
Write a Rego policy that blocks Nepali subdomains using input.host.
```

Pydantic first validates the request and scope. `classify_intent` then asks
TypeSafe to select one of the three labels. Finally, the returned SDK data is
validated as an `IntentDecision`. The example prints the expected label, actual
label, confidence, probabilities, and the two computed flags. It does not
generate or execute a Rego policy.

### Use it with another agent

```python
import asyncio

from typesafe_sdk import AsyncTypeSafeClient

from main import AgentScope, IntentClassificationRequest, classify_intent


SUPPORT_SCOPE = AgentScope(
    name="Customer support agent",
    purpose="Answer questions about product usage.",
    allowed_capabilities=("Explain product features.",),
    boundaries=("Do not change account data.",),
    valid_examples=("How do I change my notification settings?",),
    minimum_confidence=0.8,
)


async def run() -> None:
    async with AsyncTypeSafeClient() as client:
        result = await classify_intent(
            client,
            request=IntentClassificationRequest(
                user_request="How do I change my notification settings?",
                agent_scope=SUPPORT_SCOPE,
            ),
        )

        print(result.classification)
        print(result.confidence)
        print(result.probabilities)


asyncio.run(run())
```

The consuming application—not `classify_intent`—decides what happens after
these values are returned.

## Schema changes

There is no database or durable storage, so there are no migrations or resets.
The current in-memory and wire contracts are:

- `AgentScope`: `name`, `purpose`, one or more `allowed_capabilities`, optional
  `boundaries`, optional `valid_examples`, and `minimum_confidence` from 0 to 1.
- `IntentClassificationRequest`: a non-empty `user_request` and an `AgentScope`.
- TypeSafe request state: `user_request` plus the descriptive scope fields.
  `minimum_confidence` is deliberately excluded because it is applied locally.
- TypeSafe response: an `intent` choice with a label, confidence, and
  probabilities.
- `IntentDecision`: `classification`, `confidence`, `probabilities`, and
  `minimum_confidence`, plus computed `allowed` and `requires_review` fields.

The input and decision models reject unknown fields and are frozen after
creation. There is no compatibility cutover or intentional data loss.

## Project structure

- `main.py`: reusable Pydantic models and async classification function.
- `examples/policy_requests.py`: Rego-oriented scope and three sample requests.
- `intro.html`: visual, standalone explanation.

## Setup and run

Python 3.12 or later and [uv](https://docs.astral.sh/uv/) are required.

```bash
uv sync
export TYPESAFE_API_KEY="your-api-key"
uv run python -m examples.policy_requests
```

The example also reads a local `.env` file. It catches `TypeSafeAPIError` and
prints the HTTP status and request ID; other validation and runtime errors
propagate to the caller.

Users should now see one precise classification rather than an implied routing lifecycle.
