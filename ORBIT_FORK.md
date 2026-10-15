# Orbit fork of langchain-openrouter

This repository is a fork of [langchain-ai/langchain](https://github.com/langchain-ai/langchain).
Orbit uses only `libs/partners/openrouter`, pinned to tag `langchain-openrouter==0.2.9`,
with two fixes on branch `orbit-langchain-openrouter-0.2.9`. Nothing else in the
monorepo is changed.

## What's changed vs. upstream langchain-openrouter 0.2.9

**1. The serving provider reaches `response_metadata["provider"]`.**

[#39301](https://github.com/langchain-ai/langchain/pull/39301) (in 0.2.9) reads the
top-level `provider` field from OpenRouter responses. The wire carries it on every
chunk, but the `openrouter` SDK's `ChatResult` and `ChatStreamChunk` don't declare it
(OpenRouter's OpenAPI spec omits it), so `model_dump()` drops it and the field never
arrives. #39301's tests passed because they fed raw dicts, never SDK models.

`_routed_provider` falls back to the selected endpoint in `openrouter_metadata`,
which the SDK does model. It is present when the request opts in with
`x_open_router_metadata="enabled"`; when streaming it arrives on the final chunk
(OpenRouter's router-metadata docs; confirmed live 2026-10-15 for every OpenRouter
chat model Orbit wires).

**2. Mid-stream errors raise.**

OpenRouter sends a mid-stream error as a chunk with a top-level `error` *and* a
`choices` entry with `finish_reason: "error"` (OpenRouter errors docs, "Mid-stream
errors"). `_stream`/`_astream` checked `error` only on chunks without choices, so
every documented mid-stream error ended the stream as if it had completed. The
check now runs before the choices branch, and the message carries
`error.metadata.error_type` / `provider_code` when the SDK models them
(`openrouter>=0.11`).

Tests: `tests/unit_tests/test_chat_models.py::TestSdkResponseModels` parses every
fixture through the SDK's own models. Five of its tests fail on stock 0.2.9.

## Orbit tag

`langchain-openrouter-0.2.9.post1`, a PEP 440 post-release of upstream `0.2.9`
(same convention as Orbit's fastmcp fork, `v2.14.4.post1`).

```
langchain-openrouter @ git+https://github.com/ajram23/langchain@langchain-openrouter-0.2.9.post1#subdirectory=libs/partners/openrouter
```

## When to retire this fork

Switch back to PyPI `langchain-openrouter` once an upstream release surfaces the
serving provider through the SDK models and raises on `finish_reason: "error"`
chunks. Re-check both with the `TestSdkResponseModels` tests before switching.
