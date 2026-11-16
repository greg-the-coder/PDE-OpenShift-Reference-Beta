# Architecture decisions (ADRs)

Concise, current decisions for this reference architecture. One entry per decision:
what we picked, why, and the alternative we rejected. (Distilled from the RH Summit
2026 build — we keep the conclusions, not the 900‑line saga.)

> Status key: ✅ ratified 2026‑11‑02.

## D1 — Single IdP: Keycloak (RHBK) only ✅
**Picked:** Keycloak/RHBK as the one identity provider for Coder (and, in‑cluster,
for Git; Grafana if/when D8 lands). Users and groups are provisioned
**just‑in‑time at OIDC login** (Coder group/role sync, Gitea auto‑registration) —
no SCIM push. **Why:** drops dual‑IdP + group‑sync complexity; GitHub OAuth
isn't reachable air‑gapped. **Rejected:** dual‑IdP (Keycloak + GitHub).

## D2 — In‑cluster Git: Gitea ✅
**Picked:** SCM runs **in the cluster**, as **Gitea** (Helm; lightest option). **Why:** the prior EC2‑Omnibus GitLab caused
a cloud‑init systemd deadlock, a private‑subnet + edge‑Route cutover, and an SSM‑PAT
dance — none of it air‑gap‑able or repeatable. **Rejected:** GitLab‑on‑EC2;
GitLab‑operator (heavier; Gitea chosen in P2).

## D3 — Model serving via OpenShift AI (RHOAI) ✅
**Picked:** RHOAI **KServe + vLLM ServingRuntime** (`ServingRuntime` + `InferenceService`).
**Why:** RHOAI *is* the Red Hat sovereign‑AI story; replaces 6 hand‑rolled vLLM
Deployments and their drift. **Rejected:** hand‑rolled vLLM Deployments/MachineSet zoo.

## D4 — Default model: Qwen2.5‑7B‑Instruct on 1× A10G `g5.2xlarge` ✅
**Picked:** the model/GPU we actually validated — `Qwen/Qwen2.5‑7B‑Instruct`, 16K ctx,
TP‑1, `--tool-call-parser hermes`, on a single A10G. **Why:** cost‑effective default;
proven to tool‑call correctly. Llama‑3.3‑70B INT4 (TP‑2, L40S/`g6e`) is documented as
a **scale‑up** (kept at 0 for cost; documented only — no manifest in the repo yet).
**Rejected:** Llama‑70B/L40S as the default.

## D5 — No secret VALUES in the repo (it is public) — ever ✅
**Picked:** the repo contains only secret *references + generator specs*. Values come
from three places, none of them git:
1. **Operator/platform‑generated** in‑cluster — CNPG DB creds, Keycloak initial admin,
   cert‑manager TLS.
2. **Deploy‑time random‑generated in‑cluster** via **External Secrets Operator
   `Password` generators** — persona passwords + OIDC client secrets. The Keycloak
   realm **render Job** injects them into the realm; each consumer reads the *same*
   generated Secret. (OIDC client‑secret coordination with zero values in git.)
3. **Deploy‑time‑provided** — Red Hat pull secret, Coder license — via `task`/env,
   never committed. Bedrock (connected mode) uses **IRSA**, no static key.

**Why:** a public reference repo must contain *zero* secret material — not even
encrypted. **Rejected:** **Sealed Secrets / SOPS** — they still commit *ciphertext*
to permanent public history (and sealed values are useless to anyone cloning the
repo). In **connected** mode ESO may optionally be backed by AWS Secrets Manager —
still only references in git; in **air‑gap** mode ESO generators (or an in‑cluster
Vault) keep it dependency‑free. Enforcement uses **GitHub‑native secret scanning
+ push protection** (repo setting), consistent with the rest of the Coder org —
no CI secret‑scanner action is maintained in‑repo.

## D6 — Coder Agents demoed in‑product ✅
**Picked:** the headline flow is Coder Agents chats in the Coder UI (models via the
AI Gateway, configured by `task ai-provider`). The
issue→workspace **bridge** is optional / a later phase. **Why:** removes a
bespoke Go service from the critical path.

## D7 — Two workspace templates ✅
**Picked:** `workspace-basic` and `workspace-agent` (Claude Code + Codex, installed
by pinned Coder registry modules at workspace start and routed through the AI
Gateway; the image itself carries only code‑server + toolchain). An
**egress‑controlled** flavor is folded into air‑gap mode rather than a
separate template — note that air‑gap mode therefore needs the CLIs baked into
the image or mirrored (see [`air-gap.md`](air-gap.md#known-gaps)).
**Rejected:** the prior 5 templates + firewall/no‑firewall split.

## D8 — Minimal observability ✅
**Picked:** OpenShift user‑workload monitoring + 1–2 dashboards. **Rejected:** the
`coder-observability` Helm umbrella (optional add‑on later).
**Status:** not implemented yet — `manifests/observability/` is a placeholder
(the `observability` Argo app syncs an empty set).

## D9 — Five docs max ✅
**Picked:** `README.md` + `docs/{architecture,deploy,air-gap,decisions}.md`. Anything
else gets folded in or dropped. **Why:** the prior repo drifted to 33 markdown files.

## D10 — Smart Router in front of the AI Gateway ✅
**Picked:** the [vLLM Semantic Router](https://github.com/vllm-project/semantic-router)
(“Smart Router”) in **standalone** gateway mode (`manifests/smart-router/`) as the
single OpenAI-compatible endpoint Coder's AI Gateway targets, instead of pointing
the gateway straight at the model predictor. `task ai-provider` registers the
`openai` provider with `base_url = http://smart-router.smart-router.svc.cluster.local:8899/v1`
and the default Agents model `vllm-sr/auto`; Agents/Codex traffic flows
workspace → `…/api/v2/ai-gateway/openai/v1` (**logged** in AI session logs) →
Smart Router → model. Codex's Responses API (`wire_api = "responses"`) is served
by the router in-process over the chat-completions backend.

**Why:** routing policy — code vs. chat, local vs. burst, confidential-stays-local,
image → vision — lives in **one declarative ConfigMap** and can evolve without
touching Coder or the workspaces, while every request stays captured in the
gateway's session logs (governance). A small **decision model** (Vela-2.0-0.8B)
classifies each request against image / tools / confidentiality / task /
difficulty signals.

**Sovereign base = local-only.** The base config keeps the full signal/decision
framework but resolves **every** decision to the one in-cluster model (the RHOAI
Qwen2.5-7B predictor, D4) — no cloud endpoints, no external credentials, so the
air-gap story stays “local only”. The decision model runs on **CPU**
(`device: cpu`) so the router needs no GPU and coexists with the single A10G.

**Connected overlay (optional, documented — not deployed in the base):** add
Bedrock frontier/vision backends (`provider: vllm`, OpenAI-compat Bedrock
endpoint, ABSK key via ESO in a `bedrock-burst-key` Secret — no static key in
git, D5) and repoint the `multimodal` / `advanced_reasoning` decisions at them.
This is the burst-to-cloud tier.

**Perf tune (optional, documented):** move the decision model onto a
**time-sliced GPU slice** (NVIDIA GPU-operator `time-slicing-config`) — classify
latency drops from seconds to tens of ms. Not managed in the repo; CPU is the
single-A10G-deployable default.

**Rejected:** gateway → predictor directly (routing logic would have to live in
every client / template, and there would be no place to grow burst policy);
vendoring the upstream Helm chart output (the repo style is concise hand-written
raw manifests, not generated YAML).

---

### Logistics (locked 2026‑11‑02)
- Repo: canonical **`coder/PDE-OpenShift-Reference-Beta`** (public); work on fork
  `greg-the-coder/…`, PR into upstream. Argo `repoURL` = upstream.
- AWS account **`<AWS_ACCOUNT_ID>`** (target deployment account, supplied at
  deploy time), us‑east‑1. New DNS drops `rhsummit`:
  `base_domain = rh-pde.coderdemo.io` → apps at `*.apps.cluster.rh-pde.coderdemo.io`
  (confirm exact base domain before cluster provisioning).
- Air‑gap: **simulated‑on‑connected first**; fully‑disconnected path documented.
- Cloud LLM: **Bedrock kept as an optional connected‑mode provider**; air‑gap = RHOAI only.
