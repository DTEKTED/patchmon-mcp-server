# Security review: 2026-11-01

Reviewed upstream commit `bfa35271f128c3eb6d45c9644653f7659c6feb47` from
`weirdchaos/patchmon-mcp-server`. The executable `server.py` is unchanged in this
fork. This review supports a pinned, read-only, local stdio installation.

## Source findings

All five tracked files and the single-commit history were reviewed: `server.py`,
`requirements.txt`, `README.md`, `.env.example`, and `.gitignore`. There are no
submodules, symlinks, GitHub workflows, package build scripts, bundled binaries,
or installation hooks in this repository.

No obfuscation, shell execution, dynamic code loading, unrelated credential
access, or third-party exfiltration destination was found in that source. The
application reads its named configuration variables, exposes 13 MCP tools, and
makes authenticated HTTP requests to the configured PatchMon instance. HTTPX
does not follow redirects by default. TLS verification is enabled by default.
The source itself contains no analytics or telemetry client.

`delete_host` is a real destructive capability, guarded by `confirm=true` and
PatchMon's `host:delete` permission. Deploy with a dedicated token scoped to
`{"host":["get"]}` and a client allowlist containing the 12 read tools, excluding
`delete_host`. Keep the endpoint operator-controlled. Host IDs are interpolated
into paths without UUID validation; this is another reason to retain a scoped
credential. API responses, including host notes, are untrusted data.

Upstream has no LICENSE file or declared license in its repository metadata.
This fork retains upstream history and attribution and makes no new license
claim for upstream code.

## Dependency findings

Upstream permits unbounded upgrades (`fastmcp>=2.0.0`, `httpx>=0.27.0`). A candidate
FastMCP 2.14.7 environment included known FastMCP and DiskCache advisories. It was
not used for the installed server.

`requirements.lock` pins the complete Python 3.12 resolution, including FastMCP
4.1.4, MCP 2.3.0, and HTTPX 0.28.1, with distribution hashes. All 64 resolved
packages were checked against both PyPI's version-specific vulnerability data
and the OSV batch API on 2026-11-01; neither returned an advisory for this
resolution. These checks identify listed vulnerabilities and do not constitute
a line-by-line audit of all dependency source code or prove absence of malware.

Install into a dedicated virtual environment using wheels only:

```sh
uv venv .venv --python /usr/bin/python3 --no-project
uv pip install --python .venv/bin/python --require-hashes --only-binary :all: -r requirements.lock
uv pip check --python .venv/bin/python
```

Do not use the unpinned requirements file for deployment or automatically pull
new upstream code. Review changes and regenerate, audit, and test the lock
before updating.

## Runtime findings and required settings

FastMCP 4.1.4's startup banner checks PyPI for new versions by default. Its
OpenTelemetry instrumentation is inert without an exporter, but is explicitly
disabled in this deployment. Set:

```sh
FASTMCP_CHECK_FOR_UPDATES=off
FASTMCP_TELEMETRY_MODE=off
FASTMCP_SHOW_SERVER_BANNER=false
PATCHMON_VERIFY_SSL=true
```

Use local stdio, not an unauthenticated HTTP listener. Start with a minimal
environment, load only the dedicated API credential from an owner-readable
file, use Python isolated mode (`-I`), and set the working directory to this
checkout. Never load a shared homelab secrets file into the MCP process. Apply
an IP allowlist and expiry to the PatchMon credential.

## Validation performed

- MCP initialization and discovery succeeded using protocol version 2025-06-18.
- All 13 upstream tools passed against a mock API; the deletion tool was tested
  only with `confirm=false`, which made no HTTP request. The other tools made
  14 expected GET requests.
- After disabling the update check, Linux syscall tracing showed only mock
  PatchMon connections and no external connection attempts.
- The installed checkout passed all 12 enabled tools against PatchMon 2.1.6,
  including host inventory, statistics, system/network details, packages,
  reports, agent queue, notes, integrations, overview, and security-update
  filtering.
- A deletion request using the restricted token and a randomly generated,
  verified nonexistent host UUID returned HTTP 403. No existing host was used.
- A trace of the installed launch configuration showed only the configured
  PatchMon HTTPS endpoint and local DNS resolver. The process exited cleanly.
- The isolated environment passed `uv pip check`.

No suspicious application behavior was found in the reviewed source or these
tests. This conclusion is limited to the reviewed revision, dependency lock,
configuration, and exercised paths; it is not a permanent security guarantee.
