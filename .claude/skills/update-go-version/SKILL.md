---
name: update-go-version
description: Update Go version across the Tempo codebase (go.mod, tools/go.mod, Dockerfile, CI workflows, tools image tag)
allowed-tools: WebFetch, Grep, Read, Write
---

# Update Go Version

Updates the Go version across all relevant files in the Tempo codebase.

## Usage

Invoke with `/update-go-version`

## Steps to Perform

### 1. Get the version

Extract Go version from tools/Dockerfile. 

This file is updated by a Renovate workflow automatically.

`cmd/tempo/Dockerfile_debug` pins the same `golang:X.Y.Z-alpine` base image
and is updated by the same Renovate manager,
so the two Dockerfiles should always agree.
If they don't, the Renovate PRs have not all landed yet — stop and wait.

### 2. Check if go.mod files need updating

Check these files:
- `go.mod` (main module)
- `tools/go.mod` (tools module)
If the versions already match, advise user that tools/Dockerfile needs to be updated and merged first to build new image, then stop.

### 3. Update go.mod files

Update the `go X.Y.Z` directive in both:
- `go.mod` (main module)
- `tools/go.mod` (tools module)

### 4. Update TOOLS_IMAGE_TAG in build/tools.mk

Fetch the latest tools image tag from Docker Hub:
```bash
curl -s "https://hub.docker.com/v2/repositories/grafana/tempo-ci-tools/tags?page_size=5&ordering=last_updated" | jq -r '.results[0].name'
```

Update `TOOLS_IMAGE_TAG ?= main-XXXXXXX` with the latest tag.

### 5. Verify changes compile

```bash
make vendor
make build
```

### 6. Verify no version is left behind

```bash
rg --hidden -g '!vendor/**' -g '!docs/**' -e '^go \d' -e 'golang:1\.' -e 'go-version:'
```

Every hit should show the new version.
A literal `go-version:` hit is drift in itself:
CI reads the version from `go-version-file`,
so a workflow that hardcodes `go-version:` will silently stay behind.

## Files to Update

| File | What to change |
|------|----------------|
| `go.mod` | `go X.Y.Z` directive |
| `tools/go.mod` | `go X.Y.Z` directive |
| `build/tools.mk` | `TOOLS_IMAGE_TAG` value |
| `tools/Dockerfile` | `golang:X.Y.Z-alpine` base image (Renovate) |
| `cmd/tempo/Dockerfile_debug` | `golang:X.Y.Z-alpine` base image (Renovate) |

The repo pins a patch-level `go X.Y.Z` directive
and carries no `toolchain` directive in either module.
Don't add one:
a patch-level `go` directive already implies the toolchain,
and the go command drops a `toolchain` line
that isn't newer than the `go` line on the next `go mod tidy`.

CI workflows need no edit.
Every `actions/setup-go` step resolves the version through `go-version-file`
(`go.mod`, or `tools/go.mod` in `tools-tests.yml`),
so bumping the `go` directive moves CI with it.
