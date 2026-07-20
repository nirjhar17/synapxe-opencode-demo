# 03 — Configure llm-d / model serving

This is where the **model** lives: KServe `LLMInferenceService`, llm-d routing, vLLM pod, GPU, model-cache PVC.

## Goal

Serve `Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8` in namespace `ai-serving`, reachable only through the MaaS path from [02 — MaaS](02-maas.md).

## Step 1 — Namespace and model cache PVC

```bash
oc create namespace ai-serving

cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-cache
  namespace: ai-serving
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 100Gi
  storageClassName: gp3-csi   # change for your cluster
EOF
```

PVC caches weights so pod restarts do not re-download tens of GB every time.

## Step 2 — Model credentials / source (lab vs disconnected)

**Online lab (Hugging Face URI):**

```bash
oc create secret generic hf-token -n ai-serving --from-literal=token=<HF_TOKEN>
```

Model URI in the CR: `hf://Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8`

**Disconnected:** use internal S3 / OCI modelcar / preloaded PVC instead of Hugging Face at pull time. See [08 — Disconnected](08-disconnected.md). Inference hops stay the same.

## Step 3 — (Optional) llm-d Gateway in `ai-serving`

Some setups also create `llm-d-gateway` in `ai-serving` for internal routing. Lab used `data-science-gateway-class` and TLS secret for that gateway. For client traffic we standardized on **`maas-default-gateway`** in the LLMInferenceService router refs.

```bash
oc get gateway -n ai-serving
```

## Step 4 — Apply LLMInferenceService

Canonical file in this repo: [`../llminferenceservice.yaml`](../llminferenceservice.yaml)

Critical fields:

- `apiVersion: serving.kserve.io/v1alpha2` (confirm with `oc explain`)
- `spec.model.uri` / `spec.model.name`
- `spec.router.gateway.refs` → `maas-default-gateway` / `openshift-ingress`
- GPU requests/limits: `nvidia.com/gpu: "1"`
- PVC mount for model cache
- `nodeSelector` / `tolerations` for GPU nodes
- vLLM args: context length, tool-calling parser for Qwen, etc.

```bash
oc apply -f llminferenceservice.yaml
oc get llminferenceservice -n ai-serving
oc get llminferenceservice qwen3-coder -n ai-serving -o yaml
```

Wait until `READY=True`.

## Step 5 — Watch pods and GPU

```bash
oc get pods -n ai-serving -o wide
oc logs -n ai-serving -l serving.kserve.io/inferenceservice=qwen3-coder -c main --tail=100
# or describe the vLLM pod and confirm GPU allocation
```

First start can take many minutes while weights load into GPU memory.

## Step 6 — Schema / EPP gotchas (lab lessons)

1. **v1alpha1 vs v1alpha2** — older samples break; rewrite with `oc explain llminferenceservices.spec`.
2. **Custom EPP / EndpointPickerConfig** — if the EPP binary rejects your config API version, remove custom scheduler config and use controller defaults (lab fix for CrashLoopBackOff).
3. **GUI limits** — OpenShift AI UI may not expose nodeSelector, tolerations, PVC mounts, or long probe timeouts; apply YAML.

## Step 7 — Confirm path through MaaS

After Ready, verify HTTPRoute + DestinationRule + curl from [02 — MaaS](02-maas.md). Do not consider the model “done” until chat completions return 200 via the gateway.

## Mapping to slide hops

| Hop | Object |
|-----|--------|
| Gateway | `maas-default-gateway` |
| URL rewrite | HTTPRoute for `qwen3-coder` |
| Auth | AuthPolicy |
| Protocol | DestinationRule `h2UpgradePolicy: DO_NOT_UPGRADE` |
| Schedule | LLMInferenceService / InferencePool / llm-d |
| Infer | vLLM pod + GPU |

## Next

→ [04 — MCP gateway full flow](04-mcp-gateway.md)
