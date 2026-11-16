# manifests/smart-router

Smart Router — the [vLLM Semantic Router](https://github.com/vllm-project/semantic-router)
in **standalone** gateway mode (ADR D10). An OpenAI-compatible front door that
classifies each request with an on-CPU decision model and routes it to the right
backend. Synced by the `smart-router` Argo Application
(`gitops/apps/smart-router.yaml`, sync-wave 3 — after `model-serving`) into the
`smart-router` namespace.

| File | Purpose |
|------|---------|
| `namespace.yaml` | `smart-router` namespace. |
| `rbac.yaml` | `ServiceAccount` + `Role`/`RoleBinding` for self-write of its config ConfigMap, plus the `nonroot-v2` SCC grant (uid/gid 65532). |
| `config.yaml` | `ConfigMap` with the v0.5 router config: listener `:8899`, Responses API on, on-CPU Vela decision model, one `local` model → the RHOAI predictor, full signal/decision framework (all routes → `local`). |
| `deployment.yaml` | CPU-only `Deployment` (`vllm-sr`, not `-cuda`) + `Service` (`:8899` http, `:9190` metrics). |

## Why it exists

Coder's AI Gateway needs a single OpenAI-compatible endpoint. Pointing it at the
Smart Router (instead of straight at the model) means **routing policy lives in
one declarative place** and can grow — code vs. chat, local vs. burst,
confidential-stays-local — without touching Coder or the workspaces. Every
request is still classified against the full signal set (image / tools /
confidentiality / task / difficulty); the sovereign base just resolves all of
them to the single in-cluster model.

## How Coder reaches it

`task ai-provider` registers the Coder AI Gateway `openai` provider with
`base_url = http://smart-router.smart-router.svc.cluster.local:8899/v1` and the
default Agents model `vllm-sr/auto` (see [`ai-providers/`](../../ai-providers/)).
Agents and Codex therefore call `…/api/v2/ai-gateway/openai/v1` → the gateway
(request **logged** in AI session logs) → the Smart Router → the model. Codex
uses the Responses API (`wire_api = "responses"`), which the router serves
in-process over the chat-completions backend.

## Sovereign base vs. connected overlay

**Base (air-gap-clean):** one model, `local`, pointing at the RHOAI Qwen2.5-7B
predictor (`qwen25-7b-predictor.rhoai-serving.svc:8080`). No cloud endpoints, no
external credentials. Every decision resolves to `local`.

**Connected overlay (optional, documented — not in the base path):** add Bedrock
frontier/vision models as extra `backend_refs` (`provider: vllm`, OpenAI-compat
Bedrock endpoint, ABSK key via ESO in a `bedrock-burst-key` Secret) and repoint
the `multimodal` and `advanced_reasoning` decisions at them. This is the
burst-to-cloud tier; it is deliberately excluded from the sovereign base so the
air-gap story stays "local only" (ADR D10, and see [`air-gap.md`](../../docs/air-gap.md)).

## Decision model placement

The Vela-2.0-0.8B decision model runs on **CPU** (`device: cpu`) so the router
coexists with the single A10G that serves the model. Classification on CPU is
slower than on GPU (seconds vs. tens of ms, scaling with prompt size). Moving it
onto a **time-sliced GPU slice** is an optional performance tune (ADR D10); it
requires an NVIDIA GPU-operator `time-slicing-config` and is not managed here.

## Runtime config writes

The router can persist config edits back to its own ConfigMap
(`VLLM_SR_K8S_CONFIGMAP_*` env), which is why `rbac.yaml` grants it
get/update/patch on `smart-router-config`. Git remains the source of truth: Argo
`selfHeal` reverts out-of-band edits, so change routing policy by committing to
`config.yaml`.
