# Synapxe Private AI Coding Assistant - Deployment Checkpoint
## Date: July 3, 2026

## Objective
Deploy Manu Joy's Private AI Coding Assistant architecture on existing ROSA cluster,
integrating our custom skills (reports, documents, slides) and MCP configurations.
Goal: Present to Synapxe as an agentic application with zero data egress using
internally hosted open-weight models.

## Architecture Stack
- ROSA HCP 4.21 on AWS (us-east-2)
- OpenShift AI (RHOAI) 3.3+
- KServe (Headed mode) + llm-d intelligent routing
- vLLM serving Qwen3-Coder-30B-A3B-Instruct-FP8
- NVIDIA L40S GPU (g6e.2xlarge)
- Dev Spaces + OpenCode (planned)

## Cluster Access
- Console: https://console-openshift-console.apps.rosa.rhoai-lab.orwi.p3.openshiftapps.com
- API: https://api.rhoai-lab.orwi.p3.openshiftapps.com:443
- Login: cluster-admin / RpiU4-g75Wh-5piMJ-ZXVoo
- RHOAI Dashboard: https://rh-ai.apps.rosa.rhoai-lab.orwi.p3.openshiftapps.com

## What Was Done Today

### Step 1: GPU Machine Pool (DONE)
- Created machinepool `gpu-l40s` with g6e.2xlarge instance (NVIDIA L40S, 48GB VRAM)
- Node came up and joined the cluster

### Step 2: NVIDIA GPU Operator (DONE)
- Installed NVIDIA GPU Operator
- Applied ClusterPolicy CR (gpu-cluster-policy)
- Node Feature Discovery (NFD) is a dependency, confirmed working

### Step 3: Model Serving - Self-Hosted Open-Weight Model (IN PROGRESS)

#### Step 3a: Patched KServe to Headed mode (DONE)

```
oc patch dsc default-dsc --type=merge -p '{"spec":{"components":{"kserve":{"rawDeploymentServiceConfig":"Headed"}}}}'
```

#### Step 3b: Created ai-serving namespace (DONE)

```
oc create namespace ai-serving
```

#### Step 3c: Created model-cache PVC (DONE)
- 100Gi gp3-csi PVC named `model-cache` in ai-serving
- Used for persistent model storage at /mnt/models

#### Step 3d: Created HuggingFace token secret (DONE)

```
oc create secret generic hf-token --namespace=ai-serving --from-literal=token=<token>
```

#### Step 3e: Created TLS secret for llm-d gateway (DONE)
- Generated self-signed cert locally with openssl (CN=llm-d-gateway, SAN=full DNS)
- Created secret: llm-d-gateway-tls

```
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -keyout /tmp/tls.key -out /tmp/tls.crt \
  -subj "/CN=llm-d-gateway" \
  -addext "subjectAltName=DNS:llm-d-gateway-data-science-gateway-class.ai-serving.svc.cluster.local"

oc create secret tls llm-d-gateway-tls --namespace=ai-serving --cert=/tmp/tls.crt --key=/tmp/tls.key
```

#### Step 3f: Created llm-d Gateway (DONE)
- Gateway `llm-d-gateway` using `data-science-gateway-class`
- Programmed: True, got ELB address
- Internal DNS: llm-d-gateway-data-science-gateway-class.ai-serving.svc.cluster.local

#### Step 3g: LLMInferenceService (NEEDS FIX - apply YAML below)

##### Issues encountered and resolved:

1. v1alpha1 vs v1alpha2 schema mismatch
   - Manu's repo used v1alpha1, cluster has v1alpha2
   - Key changes: modelSpec->model, routerSpec->router,
     gatewayRef->gateway.refs[], workloadSpec.template.spec->template
   - Fixed by rewriting YAML to v1alpha2 schema using `oc explain`

2. EPP EndpointPickerConfig API version mismatch
   - Our config used `epp.llm-d.ai/v1alpha1` kind `EndpointPickerConfig`
   - EPP binary on RHOAI 3.3 doesn't recognize this API version
   - Fix: Remove custom scheduler.config, let controller use defaults

##### Model server status (before fix):
- vLLM pod: Running 1/1, ~33.5 GB memory (model loaded on GPU)
- EPP scheduler: CrashLoopBackOff due to config API mismatch
- Model cached on PVC (should load fast on next deploy)

##### Endpoints (once Ready):
- External: https://a6eb79e3b528f41cc898877333a495b4-533687947.us-east-2.elb.amazonaws.com/ai-serving/qwen3-coder
- Internal: https://llm-d-gateway-data-science-gateway-class.ai-serving.svc.cluster.local/ai-serving/qwen3-coder

## Key Learnings

- Always check CRD API version with `oc explain <resource>.spec` before applying YAMLs from external repos
- RHOAI GUI cannot set nodeSelector, tolerations, PVC mounts, probe timeouts, or EPP scorer weights
- v1alpha2 flattened the LLMInferenceService schema significantly
- Permission denied warnings during HF model download are cosmetic (non-root container)
- EPP config API version differs between llm-d releases

## Remaining Steps
- [ ] Fix EPP scheduler (apply YAML without custom scheduler config)
- [ ] Verify LLMInferenceService Ready: True
- [ ] Install Dev Spaces operator + CheCluster
- [ ] Build custom OpenCode container image with model config
- [ ] Create skills Git repo (clean-notes, create-report, create-slides, sa-reviewer)
- [ ] Create devfile (OpenCode image + auto-clone skills repo)
- [ ] End-to-end test
- [ ] Create demo cheat sheet for Synapxe meeting (week of July 13)

## Reference
- Manu Joy's repo: https://github.com/manujoy7/Private_AI_Coding_Assistant
- Previous OpenCode/DevSpaces chat: 3a704b6a-7f9d-4f68-91cf-8d05c22df4cb
- This session chat: 4a7a8e85-e5f7-464f-9e8f-57bbb3e5cbaf
