# 02 — Configure MaaS (Model-as-a-Service front door)

MaaS is **not the model**. It is the shared Envoy front door that clients (OpenCode) call for OpenAI-compatible inference.

## What you are building

```
Client (OpenCode)
  → HTTPS :443
  → maas-default-gateway (Envoy)
  → HTTPRoute (path per model)
  → AuthPolicy (checkpoint)
  → DestinationRule (TLS + HTTP/1.1)
  → llm-d / KServe / vLLM
```

See diagram: [`../diagrams/maas-to-model-request-flow.png`](../diagrams/maas-to-model-request-flow.png)

## Step 1 — Confirm the MaaS / data-science gateway exists

On OpenShift AI, the platform typically provides a gateway class and a default MaaS gateway:

```bash
oc get gatewayclass
oc get gateway -A | grep -iE 'maas|data-science'
```

Lab objects:

```bash
oc get gateway maas-default-gateway -n openshift-ingress -o yaml
```

Clients use either:

- Internal: `https://maas-default-gateway-data-science-gateway-class.openshift-ingress.svc.cluster.local/...`
- External ELB hostname (from Gateway status)

## Step 2 — Point the model router at MaaS

In `LLMInferenceService`, set the router gateway refs to `maas-default-gateway` (see [03 — llm-d](03-llmd-model-serving.md) and root `llminferenceservice.yaml`):

```yaml
spec:
  router:
    gateway:
      refs:
      - name: maas-default-gateway
        namespace: openshift-ingress
    route: {}
```

Applying the LLMInferenceService causes KServe/llm-d to create/manage the **HTTPRoute** for that model (path prefix like `/ai-serving/qwen3-coder`).

```bash
oc get httproute -n ai-serving
oc get httproute qwen3-coder-kserve-route -n ai-serving -o yaml
```

### What HTTPRoute does

- Match: `/ai-serving/qwen3-coder/...`
- Rewrite toward backend: `/v1/...` (what vLLM understands)
- One gateway hostname, many models via unique prefixes

## Step 3 — AuthPolicy (auth checkpoint)

Auth lives as a Kuadrant/Authorino `AuthPolicy` on the route. In the lab demo it is open so the path is easy to show:

```bash
oc get authpolicy -n ai-serving
oc get authpolicy qwen3-coder-kserve-route-authn -n ai-serving -o yaml
```

Annotation on the LLMInferenceService (lab):

```yaml
metadata:
  annotations:
    security.opendatahub.io/enable-auth: "false"
```

Production: turn auth on (API key / OIDC / mTLS) — same hop, different policy. Do not remove the control point.

## Step 4 — DestinationRule (protocol compatibility)

Envoy prefers HTTP/2 to backends. vLLM (uvicorn) speaks HTTP/1.1. Without an explicit rule you often see `503` / `http2.remote_reset` even when the pod is healthy.

Apply a DestinationRule on the **workload Service host** in `ai-serving`:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: qwen3-coder-kserve-workload-svc
  namespace: ai-serving
spec:
  host: qwen3-coder-kserve-workload-svc.ai-serving.svc.cluster.local
  trafficPolicy:
    tls:
      mode: SIMPLE
      # use cluster service CA as required by your mesh setup
    connectionPool:
      http:
        h2UpgradePolicy: DO_NOT_UPGRADE
```

```bash
oc apply -f destinationrule-qwen.yaml
oc get destinationrule -n ai-serving
```

**Meaning:** re-encrypt toward the backend if required, and **force HTTP/1.1** (`DO_NOT_UPGRADE`).

## Step 5 — Smoke test through MaaS (not direct to the pod)

```bash
# From a pod that trusts cluster TLS (or with -k for lab)
curl -k -sS -X POST \
  "https://maas-default-gateway-data-science-gateway-class.openshift-ingress.svc.cluster.local/ai-serving/qwen3-coder/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8","messages":[{"role":"user","content":"ping"}],"max_tokens":32}'
```

Expect HTTP 200 and a JSON completion.

## Why not point OpenCode at the vLLM Service?

You can in a lab. You lose: one stable URL, multi-model path prefixes, AuthPolicy, DestinationRule consistency, and healthy replica scheduling. MaaS is the platform front door.

## Next

→ [03 — Configure llm-d / model serving](03-llmd-model-serving.md)
