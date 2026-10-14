# Bug Report: Lightspeed/vLLM Tool Calls Emitted as Text, Not Parsed Into Structured `tool_calls`

**Status**: Open, unresolved. Workaround: none found; demo narrates the diagnosis as a known issue if it comes up.
**Component**: OpenShift Lightspeed (lightspeed-service-api) talking to vLLM's `/v1/responses` endpoint, model served with `--tool-call-parser=hermes`.
**Affected versions confirmed**: Lightspeed operator v1.1.5, vLLM image `registry.redhat.io/rhaiis/vllm-cuda-rhel9:3.3.0`, model `RedHatAI/Qwen3-8B-FP8-dynamic`, RHOAI 3.5.1.
**Severity**: Medium — blocks the "apply the fix" half of `demo/lightspeed-demo`; diagnosis-only interactions aren't affected since the model still reasons correctly, it just can't execute the MCP tool call it proposes.

## Summary

Lightspeed's agent loop correctly builds the request (after fixing token-budget issues — see the `contextWindowSize`/`toolBudgetRatio` comments in `demo/lightspeed-demo/manifests/olsconfig.yaml`) and the model correctly decides to call an MCP tool. But instead of the tool call coming back as a structured field vLLM/Lightspeed can execute, the model emits it as literal text in its answer:

```
<think>

</think>

<tool_call>
{"name": "resources_list", "arguments": {"apiVersion": "apps/v1", "kind": "Deployment", "namespace": "lightspeed-demo"}}
</tool_call>
```

Lightspeed's own log confirms it did not recognize this as a tool call:
```
Tool loop iteration 1/5 outcome=model_finished_without_tools. Token budget after round 1: ...
```

The `<tool_call>...</tool_call>` tags are Qwen/Hermes' native chat-template syntax for function calling — this is exactly what `--tool-call-parser=hermes` exists to intercept and convert into vLLM's structured `tool_calls` response field. That conversion isn't happening here.

## Environment

| Field | Value |
|---|---|
| Cluster | `ocp.5msvc.sandbox5559.opentlc.com`, AWS |
| RHOAI | 3.5.1 |
| Lightspeed operator | v1.1.5 |
| vLLM image | `registry.redhat.io/rhaiis/vllm-cuda-rhel9:3.3.0` |
| Model | `RedHatAI/Qwen3-8B-FP8-dynamic`, served via `LLMInferenceService` with `--max-model-len=8192 --enable-auto-tool-choice --tool-call-parser=hermes` |
| MCP server | `ocp-mcp-server` (`registry.redhat.io/openshift-mcp-beta/openshift-mcp-server-rhel9:0.2`), write-enabled |

## Working theory

Lightspeed talks to vLLM via the newer OpenAI **Responses API** (`POST /v1/responses`), not the classic `/v1/chat/completions` endpoint. `--tool-call-parser` support in vLLM was built around `/v1/chat/completions`'s `tools`/`tool_choice` request fields and `tool_calls` response field. It's plausible the Responses API code path (`vllm/entrypoints/openai/serving_responses.py`) either doesn't thread tool definitions through the same parser, or doesn't invoke the hermes parser on the output at all, letting the model's raw chat-template tool-call syntax pass through as plain assistant text instead.

This is a theory, not confirmed by reading vLLM's `serving_responses.py` source directly — the investigation stopped here to timebox the session.

## Steps to Reproduce

1. Deploy `demo/lightspeed-demo` against a `qwen3-8b`-class model with `--tool-call-parser=hermes` and MCP tools registered (`ocp-mcp-server`).
2. In the Lightspeed console chat, ask a question that requires tool use, e.g. "Why are the pods in `<namespace>` failing?"
3. Observe the response contains a literal `<tool_call>{...}</tool_call>` block as text, and no actual MCP tool invocation happens (no corresponding call in `ocp-mcp-server`'s logs, no cluster state inspected).

## What was ruled out before landing here

- Not a token-budget issue — confirmed by log line `Token budget after round 1: ... total_used=4545/8192 remaining=1599` showing plenty of headroom when this reproduced.
- Not an MCP server connectivity issue — `ocp-mcp-server`'s tool list was successfully fetched and included in the prompt (visible in the `tool_defs=3904` accounting and the earlier raw MCP `tools/list` response in `lightspeed-service-api` logs).
- Not a write-permission/RBAC issue — the loop never got far enough to attempt the actual tool call; it stopped at the text-emission stage.

## Recommended Next Steps

- Check vLLM's GitHub issues/changelog for `/v1/responses` + `--tool-call-parser` support status as of `v3.3.0` (rhaiis image tag) — may simply be unimplemented for this endpoint.
- Check whether OLSConfig/Lightspeed has a setting to force the classic `/v1/chat/completions` endpoint instead of `/v1/responses`.
- Try a non-quantized or different tool-calling-certified model to see if this is model-specific or endpoint-specific.
- File upstream with Red Hat / vLLM project if confirmed as an endpoint-level gap, referencing this report.

## Related

- `demo/lightspeed-demo/manifests/olsconfig.yaml` — `contextWindowSize`, `toolBudgetRatio`, `introspectionEnabled: false`, `querySystemPrompt: "/no_think"` fixes made in the same session, required before this issue became visible.
- `docs/reference/BUG-REPORT-OGX-REGISTRY-AUTH-401.md` — an unrelated bug found in the same engagement (OGX operator registry auth), not to be confused with this one.
