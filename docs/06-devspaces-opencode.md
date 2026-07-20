# 06 — Bake OpenCode into Dev Spaces

How the IDE workspace is created from git and how OpenCode is pre-wired to MaaS + MCP.

## Mental model

```
Git repo (this project)
  ├── Dockerfile.opencode     → custom image with OpenCode binary
  ├── devfile.yaml            → workspace recipe (clone, env, postStart)
  ├── .opencode/skills/       → skills cloned with the project (PVC)
  └── AGENTS.md               → shared agent instructions
         ↓
Dev Spaces CheCluster
         ↓
DevWorkspace pod (UDI + OpenCode)
         ↓
opencode.json written at postStart → model + MCP URLs
         ↓
OpenCode Web UI on :4096 (Route)
```

## Step 1 — Install Dev Spaces

OperatorHub: **Red Hat OpenShift Dev Spaces** (`devspacesoperator`) + DevWorkspace Operator.

```bash
oc get checluster -A
oc get checluster devspaces -n openshift-operators -o yaml
```

Optional for long demos (disable idle stop):

```bash
oc patch checluster devspaces -n openshift-operators --type=merge \
  -p '{"spec":{"devEnvironments":{"secondsOfInactivityBeforeIdling":-1}}}'
# restore later, lab default was 1800
```

## Step 2 — Build / publish the OpenCode image

File: [`../Dockerfile.opencode`](../Dockerfile.opencode)

- Base: `registry.redhat.io/devspaces/udi-rhel8:latest`
- Install OpenCode CLI to `/usr/local/bin/opencode`
- `pip3 install python-pptx python-docx` (for skills)
- Lab image: `quay.io/njajodia/devspaces-opencode:latest`

```bash
podman build -f Dockerfile.opencode -t quay.io/<org>/devspaces-opencode:latest .
podman push quay.io/<org>/devspaces-opencode:latest
```

Disconnected: mirror this image into your private registry and change `devfile.yaml` image field.

## Step 3 — Devfile is the workspace recipe

File: [`../devfile.yaml`](../devfile.yaml)

It defines:

| Piece | What it does |
|-------|----------------|
| `projects[].git` | Clones this repo into `/projects` |
| `components[].container.image` | Uses the baked OpenCode UDI image |
| `env` | `OPENAI_BASE_URL` / `VLLM_ENDPOINT` → MaaS path for Qwen |
| `endpoints` | Public HTTPS endpoint for OpenCode web `:4096` |
| `postStart` | Writes config, downloads VS Code extension, starts `opencode web` |

### What goes in `opencode.json` (you decide)

OpenCode does not invent config. **You** author JSON (via the `cat > .../opencode.json` heredoc in the devfile) using OpenCode’s schema:

- **provider / model** — MaaS base URL + model id  
- **mcp** — remote gateway URL  

Skills stay in `.opencode/skills/` (git). Behavior rules stay in `AGENTS.md`.

After workspace start:

```bash
oc exec -n <devspaces-ns> deploy/<workspace-deploy> -- \
  cat /home/user/.config/opencode/opencode.json
```

## Step 4 — Create the workspace from git

In Dev Spaces UI: create workspace from  
https://github.com/nirjhar17/synapxe-opencode-demo.git  

Or apply a DevWorkspace that points at that repo/devfile.

```bash
oc get dw -A
oc get dw opencode-private-ai -n cluster-admin-devspaces -o yaml
oc get pods,svc,route -n cluster-admin-devspaces
```

OpenCode web Route (lab pattern): `*-opencode-web` on port **4096**.

## Step 5 — Skills persistence

Skills under `.opencode/skills/` are **in git**, cloned onto the workspace PVC. Restarting the workspace keeps them. That is why 100 users sharing one repo get the same baseline skills/`AGENTS.md`.

Personal overrides: local uncommitted files or `~/.config/opencode/` (not shared unless you design that).

## Step 6 — Show customers (commands)

```bash
# Platform
oc get checluster devspaces -n openshift-operators -o yaml

# This workspace
oc get dw -n cluster-admin-devspaces -o yaml
oc get route -n cluster-admin-devspaces | grep opencode-web

# OpenCode wiring
oc exec -n cluster-admin-devspaces deploy/<ws> -- \
  ls /projects/synapxe-opencode-demo/.opencode/skills
oc exec -n cluster-admin-devspaces deploy/<ws> -- \
  cat /home/user/.config/opencode/opencode.json
```

## Next

→ [07 — Verification](07-verification.md)
