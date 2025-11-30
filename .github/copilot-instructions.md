# Go Release Action - AI Agent Instructions

## Project Overview
This is a **GitHub Action** packaged as a Docker container that builds and releases Go binaries for multiple platforms (GOOS/GOARCH) and uploads them as GitHub Release assets. The action runs inside a Debian container with Go, upx, and custom upload tools pre-installed.

## Architecture & Core Components

### Execution Flow
1. **GitHub Action Entry** ([action.yml](../action.yml)) - Defines 26+ input parameters passed to Docker container as positional args
2. **Docker Container** ([Dockerfile](../Dockerfile)) - Debian slim-based image with Go toolchain, upx, and github-assets-uploader pre-installed
3. **entrypoint.sh** - Sources `setup-go.sh` then calls `release.sh`
4. **setup-go.sh** - Installs Go from URL (supports versions like `1.21`, `1.21.1`, URLs, or `go.mod` paths)
5. **release.sh** - The core build/release logic (~250 lines)

### Key Design Patterns

#### Input Parameter Convention
All GitHub Action inputs arrive as environment variables prefixed with `INPUT_` (uppercase):
- `goos` → `INPUT_GOOS`
- `binary_name` → `INPUT_BINARY_NAME`
- Check these in [release.sh](../release.sh#L5-L15)

#### Asset Naming Pattern
Default format: `${BINARY_NAME}-${RELEASE_TAG}-${GOOS}-${GOARCH}`
- Microarchitecture variants appended: `-v3` for amd64/v3, `v7` for arm/v7
- Overridable via `asset_name` input (user must include `-${{ matrix.goos }}-${{ matrix.goarch }}` for matrix builds)

#### Build Artifacts Isolation
Creates temporary `build-artifacts-<timestamp>` folder to avoid conflicts in parallel matrix builds:
```bash
BUILD_ARTIFACTS_FOLDER=build-artifacts-$(date +%s)
```

#### Multi-Binary Support
When `multi_binaries: true`:
- Leverages Go's multi-package build: `go build -o dir ./cmd/...` or `go build -o dir ./cmd/app1 ./cmd/app2`
- All binaries packaged together in one archive
- See [test/multi-binaries/](../test/multi-binaries/) for examples

## Critical Developer Workflows

### Testing the Action
The action tests **itself** via [.github/workflows/autotest.yml](workflows/autotest.yml):
```yaml
- uses: ./  # Tests the action from current checkout
  with:
    project_path: ./test/
    binary_name: testmain
    pre_command: go mod init localtest  # Test project has no go.mod
```
**Test matrix covers**: 9+ GOOS/GOARCH combinations, goversion variations, Makefile builds, compression modes, multi-binaries

### Building Locally
Cannot run locally as a standard script - designed for GitHub Actions environment. To test:
1. Create a test workflow in `.github/workflows/`
2. Use `uses: ./` to reference local action
3. Push to branch and check Actions tab

### Makefile Build Support
Special handling when `build_command` starts with `make`:
- Executes `make` directly with GOOS/GOARCH env vars
- Ignores `build_flags` and `ldflags` (assumes they're in Makefile)
- Expects binary at `${project_path}/${BINARY_NAME}${EXT}` post-build
- Example: [test/Makefile](../test/Makefile) uses `-ldflags` to inject git commit/build time

## Project-Specific Conventions

### Go Version Handling ([setup-go.sh](../setup-go.sh))
Accepts multiple formats via `goversion` input:
- Empty string → fetches latest from `https://go.dev/VERSION?m=text`
- `1.21` → fetches latest 1.21.x patch version
- `1.21.1` → fetches exact version
- `https://...tar.gz` → downloads from URL (must be linux-amd64 package)
- `go.mod` or `./go.mod` → parses Go version from `go` directive

### Compression Strategy
`compress_assets` input controls packaging:
- `auto`/`TRUE` (default): `.zip` for Windows, `.tar.gz` for others
- `zip`: Forces `.zip` for all platforms
- `off`/`FALSE`: No compression, raw binary only

### Upload Retry Logic
Uses custom `github-assets-uploader` tool (not standard `gh` CLI):
- Built for robustness with configurable retry (default: 3 attempts)
- Supports GitHub Enterprise via `-baseurl` flag
- Handles overwrite via `-overwrite` flag

### Error Prevention Pattern
Script uses `#!/bin/bash -eux`:
- `-e`: Exit on first error
- `-u`: Error on undefined variables
- `-x`: Print each command (useful for GitHub Actions logs)

## External Dependencies

### Pre-installed Tools (Dockerfile)
- **upx** v4.0.0+ - Executable compression (optional via `executable_compression: upx`)
- **github-assets-uploader** v0.13.0 - Custom uploader replacing `gh` CLI
- **jq** - Parses GitHub event JSON (`GITHUB_EVENT_PATH`)

### GitHub Context Variables Used
- `GITHUB_REPOSITORY` - Derives default binary name
- `GITHUB_REF` - Extracts release tag
- `GITHUB_EVENT_PATH` - Fallback for release tag when `GITHUB_REF` empty
- `GITHUB_WORKSPACE` - Working directory (added to git safe.directory)
- `GITHUB_TOKEN` - Required for asset uploads
- `GITHUB_SERVER_URL` - Detects GitHub Enterprise

## Common Pitfalls

1. **Matrix builds with custom `asset_name`**: Must include platform suffix manually:
   ```yaml
   asset_name: myapp-${{ matrix.goos }}-${{ matrix.goarch }}
   ```

2. **`go.mod` in test/**: Test projects intentionally lack `go.mod` - use `pre_command: go mod init localtest` in workflows

3. **Makefile builds**: Binary must be output to `${project_path}/` (not subdirectories)

4. **Compression with multiple binaries**: All binaries packed together - no per-binary archives

5. **`extra_files` with `multi_binaries`**: Files copied differently - check `MULTI_BINARIES` branching in [release.sh](../release.sh#L94-L115)

## Key Files Reference
- [action.yml](../action.yml) - Input/output schema (26 inputs, 1 output)
- [release.sh](../release.sh) - Core build/package/upload logic
- [setup-go.sh](../setup-go.sh) - Go installation with version resolution
- [test/test_main.go](../test/test_main.go) - Reference test binary with ldflags example
- [.github/workflows/autotest.yml](workflows/autotest.yml) - Comprehensive test suite
