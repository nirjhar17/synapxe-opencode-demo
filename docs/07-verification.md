# 07 — Verification commands

Use this checklist after install or before a customer demo.

## A. Operators and CRDs

```bash
oc get csv -A | grep -iE 'devspace|rhods|gpu|servicemesh|kuadrant|authorino|rhcl|devworkspace'
oc explain llminferenceservices.spec
oc explain mcpserverregistrations.spec
```

## B. MaaS + llm-d (inference)

```bash
oc get gateway maas-default-gateway -n openshift-ingress
oc get httproute -n ai-serving
oc get authpolicy -n ai-serving
oc get destinationrule -n ai-serving
oc get llminferenceservice -n ai-serving
oc get pods -n ai-serving -o wide
```

Curl through MaaS (lab):

```bash
curl -k -sS -o /dev/null -w "%{http_code}\n" -X POST \
  "https://maas-default-gateway-data-science-gateway-class.openshift-ingress.svc.cluster.local/ai-serving/qwen3-coder/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8","messages":[{"role":"user","content":"ping"}],"max_tokens":16}'
# expect 200
```

## C. MCP gateway + server

```bash
oc get gateway mcp-gateway -n gateway-system
oc get mcpgatewayextension -A
oc get httproute -A | grep -i mcp
oc get mcpserverregistration -A
oc get mcpserverregistration minio-mcp -n minio -o yaml
oc get deploy,svc -n mcp-system
oc logs -n mcp-system deploy/mcp-gateway --tail=50
```

## D. Dev Spaces + OpenCode

```bash
oc get checluster -n openshift-operators
oc get dw -n cluster-admin-devspaces
oc get pods,route -n cluster-admin-devspaces
# Open OpenCode web Route :4096 — short prompt should return from private model
```

## E. End-to-end demo prompts (UI)

1. Short ping to the private model  
2. Show `.opencode/skills/` in the project tree  
3. `What tools and systems do you currently have access to using MCP?`  
4. List documents in MinIO drives via MCP  
5. Run a skill (`clean-notes` / `create-report` / `create-slides`)

## Pass criteria

| Check | Pass |
|-------|------|
| LLMInferenceService Ready | True |
| Chat completions via MaaS | HTTP 200 |
| DestinationRule present | `h2UpgradePolicy: DO_NOT_UPGRADE` |
| MCPServerRegistration | Ready, tools > 0 |
| OpenCode UI | Answers without public API key |
| Skills | Present under `.opencode/skills/` |

## Next

→ [08 — Disconnected notes](08-disconnected.md)
