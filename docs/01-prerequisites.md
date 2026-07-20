# 01 — Prerequisites and operators

Install these layers before MaaS, llm-d, MCP, or Dev Spaces. Order matters for GPU and gateway classes.

## Cluster baseline

- OpenShift 4.16+ (lab used ROSA ~4.21)
- `cluster-admin` (or equivalent) for operators and gateways
- At least one GPU node for the coding model (lab: `g6e.2xlarge`, NVIDIA L40S 48GB)

## Operator stack (install order)

1. **Node Feature Discovery (NFD)** — required by GPU Operator  
2. **NVIDIA GPU Operator** — makes `nvidia.com/gpu` schedulable  
3. **OpenShift Service Mesh / Gateway API pieces used by RHOAI** — Envoy `Gateway` / `HTTPRoute`  
4. **Red Hat OpenShift AI (`rhods-operator`)** — MaaS, KServe, llm-d serving  
5. **Authorino + Red Hat Connectivity Link (Kuadrant)** — AuthPolicy + MCP gateway CRDs  
6. **DevWorkspace Operator** — workspace pods  
7. **OpenShift Dev Spaces (`devspacesoperator`)** — CheCluster / browser IDE  

Upstream of OpenShift AI / RHODS: [Open Data Hub](https://opendatahub.io).

## Namespaces you will use

| Namespace | Purpose |
|-----------|---------|
| `ai-serving` | Model, PVC, LLMInferenceService, DestinationRule |
| `openshift-ingress` | `maas-default-gateway` (and related) |
| `mcp-system` | MCP controller / gateway services |
| `gateway-system` | MCP `Gateway` object (lab) |
| `minio` | MinIO + MinIO MCP server + registration |
| `openshift-operators` | Dev Spaces `CheCluster` |
| `<user>-devspaces` | Per-user DevWorkspaces |

## KServe mode (lab)

Patch DataScienceCluster so KServe uses headed raw deployment (required for this llm-d path):

```bash
oc patch dsc default-dsc --type=merge -p \
  '{"spec":{"components":{"kserve":{"rawDeploymentServiceConfig":"Headed"}}}}'
```

Confirm with:

```bash
oc get dsc default-dsc -o yaml | grep -A5 kserve
```

## GPU node readiness

```bash
oc get nodes -l node.kubernetes.io/instance-type=g6e.2xlarge
oc describe node <gpu-node> | grep -A5 Allocatable
# expect nvidia.com/gpu: 1
```

## Validate CRDs exist before applying YAML

```bash
oc explain llminferenceservices.spec
oc explain gateways.gateway.networking.k8s.io.spec
oc explain mcpserverregistrations.spec
oc explain destinationrules.networking.istio.io.spec
```

If `oc explain` fails, the operator/CRD is not installed yet.

## Next

→ [02 — Configure MaaS](02-maas.md)
