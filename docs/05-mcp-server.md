# 05 — MCP server full flow

Pattern to plug **one** tool server into the MCP gateway. Repeat per server (MinIO, OpenShift MCP, internal KB, …).

## Pattern (every server)

```
4a Deploy MCP server (Deployment + Service)
4b ReferenceGrant (if cross-namespace to Gateway)
4c HTTPRoute (parentRefs → mcp-gateway listener)
4d MCPServerRegistration (toolPrefix + targetRef → HTTPRoute)
```

Lab live example: `MCPServerRegistration/minio-mcp` in `minio`, prefix `minio_`, target HTTPRoute `minio-mcp-route`, path `/mcp`, Ready with tools.

## 4a — Deploy the MCP server

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-mcp-server
  namespace: my-namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-mcp-server
  template:
    metadata:
      labels:
        app: my-mcp-server
    spec:
      containers:
        - name: mcp-server
          image: <your-mcp-server-image>
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: my-mcp-server
  namespace: my-namespace
spec:
  selector:
    app: my-mcp-server
  ports:
    - port: 8080
      targetPort: 8080
```

For MinIO MCP: point the server at your MinIO S3 endpoint/credentials via env or Secret (do not commit secrets).

```bash
oc get pods,svc -n minio
```

## 4b — ReferenceGrant (cross-namespace only)

If the HTTPRoute namespace ≠ Gateway namespace:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-my-mcp
  namespace: gateway-system
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: my-namespace
  to:
    - group: gateway.networking.k8s.io
      kind: Gateway
```

## 4c — HTTPRoute

Wire the server Service to the MCP Gateway listener:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-mcp-route
  namespace: my-namespace
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: mcp-gateway
      namespace: gateway-system
      sectionName: mcp
  hostnames:
    - "mcp-gateway.apps.your-cluster.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /mcp
      backendRefs:
        - name: my-mcp-server
          port: 8080
```

Lab MinIO route name: `minio-mcp-route` (hostname may be internal like `minio-mcp.mcp.local` depending on design).

```bash
oc get httproute -A | grep -i mcp
oc get httproute minio-mcp-route -n minio -o yaml
```

## 4d — MCPServerRegistration

Register into the federated tool list. `toolPrefix` avoids name collisions across servers.

```yaml
apiVersion: mcp.kuadrant.io/v1alpha1
kind: MCPServerRegistration
metadata:
  name: my-mcp
  namespace: my-namespace
spec:
  toolPrefix: "mytools_"
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: my-mcp-route
    namespace: my-namespace
```

Lab MinIO:

```bash
oc get mcpserverregistration minio-mcp -n minio -o yaml
# PREFIX=minio_  TARGET=minio-mcp-route  PATH=/mcp  READY=True  TOOLS=<n>
```

**Namespace tip:** `oc get mcpserverregistration -A` finds all; without `-n` / `-A`, `oc` looks only in the current project (empty list is a common demo footgun).

## Verify the server is actually used

1. OpenCode: list MCP tools → expect `minio_*` (or your prefix).  
2. OpenCode: “List documents in both drives using MCP” → tool call + results.  
3. Cluster:

```bash
oc get mcpserverregistration -A
oc logs -n mcp-system deploy/mcp-gateway --tail=50
```

## Adding another server later

Repeat 4a–4d with a new prefix. No need to rebuild OpenCode — same remote MCP URL federates the new tools after registration becomes Ready.

## Next

→ [06 — Bake OpenCode into Dev Spaces](06-devspaces-opencode.md)
