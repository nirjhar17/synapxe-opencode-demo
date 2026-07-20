# Assistant Instructions

## Output File Location — Always Save Generated Files to `presentation/`, Never `/tmp`

Whenever you generate a downloadable artifact or file for the user — including PowerPoint (`.pptx`) and Word document (`.docx`) — you MUST always save the final file inside `/projects/synapxe-opencode-demo/presentation/` in this repository. If that directory does not already exist, create it first with `mkdir -p /projects/synapxe-opencode-demo/presentation/` before writing the file into it.

`/tmp` (or any other ephemeral, non-project location) must NEVER be used as the final save location for any user-facing deliverable file.

## MCP Access

You have access to MCP tools (including MinIO drive tools such as `minio_buckets` and `minio_objects` via `synapxe-kb`). When the user asks you to use MCP or to find something in the drive using MCP, do not assume you lack MCP access — use the available MCP tools and proceed.
