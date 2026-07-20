# 08 — Disconnected / air-gapped notes

## Inference path does not change

Slide 4 / MaaS hops stay the same offline:

OpenCode → MaaS → HTTPRoute → AuthPolicy → DestinationRule → KServe/vLLM → GPU → back.

Disconnected mode changes **how you stock** the platform (slide 3 top boxes), not the request path.

## What to replace for air-gap

| Online lab | Disconnected equivalent |
|------------|-------------------------|
| Hugging Face model URI | Internal S3 / OCI modelcar / preloaded PVC |
| Public git (GitHub) | Self-hosted source control (repo + skills) |
| open-vsx.org | Internal Open VSX mirror **or** pre-baked extensions in the image/devfile |
| quay.io / registry.redhat.io pulls | Mirrored registries (including `devspaces-opencode` image) |
| Public MCP hostname only if required | Internal MCP URL reachable inside the cluster |

## Diagram

See [`../diagrams/opencode-openshift-architecture-disconnected.png`](../diagrams/opencode-openshift-architecture-disconnected.png):

- Source control (self-hosted) — source + skills  
- Internal S3 / OCI images / PVC — model weights  
- Developer browser + Dev Spaces still provide the VS Code–like IDE  

## OpenCode / Devfile adjustments

- Point `OPENAI_BASE_URL` / `opencode.json` `baseURL` at the **in-cluster** MaaS Service DNS (already true in this repo).  
- Point MCP `url` at an **internal** MCP gateway hostname if the public apps route is unavailable.  
- Remove or replace `download-opencode-extension` curl to open-vsx; bake the VSIX into the image instead.  
- Ensure skills/`AGENTS.md` are available from self-hosted git.

## Validation offline

Same checklist as [07 — Verification](07-verification.md), but confirm:

- No egress required for chat completions  
- Model weights already on PVC/OCI/S3  
- Extension installs do not hit the public internet  
