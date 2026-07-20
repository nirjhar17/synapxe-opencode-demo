# 04 — MCP gateway full flow

The MCP gateway is the **tool front door** (separate from MaaS). Agents send MCP JSON-RPC (`tools/list`, `tools/call`) to one URL; the gateway federates registered servers.

## End-to-end picture

```
OpenCode (Dev Spaces)
  → HTTPS remote MCP URL (/mcp)
  → MCP Gateway (Envoy + MCP filters)
  → MCPServerRegistration(s)
  → HTTPRoute → MCP server Service
  → tool result back to agent
```

Transports (common):

1. **stdio** — local process MCP servers  
2. **HTTP** (SSE / streamable HTTP) — remote (what this lab uses)

Message format underneath: **JSON-RPC**.

## Stage 1 — Install the operator (Connectivity Link / Kuadrant MCP)

MCP gateway ships with **Red Hat Connectivity Link** (Kuadrant productization). Install from OperatorHub (`mcp-gateway` / Connectivity Link / related Kuadrant operators).

Expect CRDs:

```bash
oc get crd | grep -iE 'mcp|kuadrant'
oc explain mcpserverregistrations.spec
oc explain mcpgatewayextensions.spec
```

Namespaces commonly involved: `mcp-system`, `gateway-system`.

## Stage 2 — Create a Kubernetes Gateway (Envoy)

This is a normal Gateway API `Gateway` — not MCP-specific yet. Lab used `data-science-gateway-class`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: mcp-gateway
  namespace: gateway-system
spec:
  gatewayClassName: data-science-gateway-class
  listeners:
    - name: mcp
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
```

```bash
oc apply -f mcp-gateway.yaml
oc get gateway mcp-gateway -n gateway-system
# wait until PROGRAMMED=True and ADDRESS is set
```

At this point you have a reverse proxy + load balancer with **no** MCP awareness.

## Stage 3 — Attach MCPGatewayExtension

This teaches the Gateway to speak MCP (parse JSON-RPC, federate `tools/list`):

```yaml
apiVersion: mcp.kuadrant.io/v1alpha1
kind: MCPGatewayExtension
metadata:
  name: mcp-extension
  namespace: mcp-system
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: mcp-gateway
    namespace: gateway-system
    sectionName: mcp
```

```bash
oc apply -f mcp-gateway-extension.yaml
oc get mcpgatewayextension -A
```

After this: gateway understands MCP but returns an **empty** tool list until servers are registered.

## Stage 4 — Expose a stable URL for agents

Lab public hostname pattern:

`https://mcp-gateway.apps.<cluster>/mcp`

Confirm route/HTTPRoute:

```bash
oc get httproute -n mcp-system
oc get httproute mcp-gateway-route -n mcp-system -o yaml
oc get deploy,svc -n mcp-system
```

OpenCode remote MCP block (written by `devfile.yaml`):

```json
"mcp": {
  "synapxe-kb": {
    "type": "remote",
    "url": "https://mcp-gateway.apps.<cluster>/mcp",
    "enabled": true,
    "oauth": false
  }
}
```

## Stage 5 — Register servers

Each tool backend is a separate flow — see [05 — MCP server](05-mcp-server.md).

```bash
oc get mcpserverregistration -A
```

Lab example: `minio-mcp` in namespace `minio`, prefix `minio_`, Ready with tool count.

## Verify gateway federation

From OpenCode (best demo):

> What tools and systems do you currently have access to using MCP?

From cluster:

```bash
oc logs -n mcp-system deploy/mcp-gateway --tail=80
oc get mcpserverregistration -A -o wide
```

## Common gotchas

- Forgetting `MCPGatewayExtension` → plain HTTP proxy, not MCP federation  
- Registration without Ready HTTPRoute/Service → tools never appear  
- Cross-namespace routes without ReferenceGrant → attach fails  
- Confusing MaaS URL with MCP URL — different gateways  

## Next

→ [05 — MCP server full flow](05-mcp-server.md)
