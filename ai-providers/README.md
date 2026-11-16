# ai-providers

Coder AI Gateway providers and Coder Agents models. Coder 2.39 stores both in
the database only (`CODER_AI_GATEWAY_*` provider env vars are ignored), so
[`configure.py`](configure.py) sets them through the Coder API. It is
idempotent: each run creates what is missing and updates what has drifted. Run
it with `task ai-provider`; `task deploy` runs it after `push-templates`. It
needs `coder login` as an owner and an active license (AI Gateway is Premium).

| Provider (route `/api/v2/ai-gateway/<name>`) | Backend | Used by | When |
|---|---|---|---|
| `openai` | **Smart Router** (vLLM Semantic Router), `http://smart-router.smart-router.svc.cluster.local:8899/v1` — classifies + routes to the in-cluster RHOAI predictor (and, in connected mode, bursts) | Codex (Responses API) + Agents | always |
| `bedrock` | AWS Bedrock (Anthropic), us-east-1. Credentials come from the coder-server pod's IRSA role; no static keys | Claude Code | the coder SA has an IRSA role (`CODER_BEDROCK_ROLE_ARN`, `task coder-irsa`) |

Coder Agents models:

| Model | Provider | Default |
|---|---|---|
| Claude Opus 4.5 (`global.anthropic.claude-opus-4-5-20251101-v1:0`) | `bedrock` | yes, when Bedrock is enabled |
| Claude Haiku 4.5 (`global.anthropic.claude-haiku-4-5-20251001-v1:0`) | `bedrock` | |
| Smart Router — auto (`vllm-sr/auto`) | `openai` | yes, when Bedrock is not enabled (RHOAI-only) |

Overrides (environment variables): `BEDROCK_REGION`, `BEDROCK_MODEL`,
`BEDROCK_SMALL_FAST_MODEL`, `VLLM_MODEL`, `VLLM_CONTEXT_LIMIT`, and
`AI_PROVIDER_BASE_URL` (the Smart Router base URL, set via the task).

Notes:
- The gateway rewrites a model id the Bedrock provider doesn't know to the
  provider's `model` (Opus). Keep the templates' `anthropic_model` /
  `anthropic_available_models` in line with the models above. In account
  973294445171, Sonnet 4.5 is denied as "Legacy".
- The Bedrock IAM role is `terraform/` `aws_iam_role.coder_bedrock`
  (`enable_bedrock = true`). It trusts `system:serviceaccount:coder:coder` via
  the cluster OIDC issuer.
