# REA reverse-engineering setup

This repo's coding agent is wired to [REA](https://github.com/morluto/rea) ("Reverse Engineer
Anything"), a local MCP server for binary and application analysis. Ghidra is configured as its
native decompiler, so the agent can open ELF/PE/Mach-O targets and produce decompiled C.

Last verified: 2026-11-15, `rea doctor` reported `healthy: true`, `environment_healthy: true`.

## What is installed

| Component | Version | Location |
| --- | --- | --- |
| REA CLI + MCP server (`rea-agents`) | 6.9.0 | npx cache: `~/.npm/_npx/af752b90495482d7` |
| REA investigation skill | catalog 62, 255 tools | `~/.agents/skills/reverse-engineer-anything` |
| Ghidra (headless provider) | 12.1.4 PUBLIC | `~/.local/share/rea/engines/ghidra_12.1.4_PUBLIC` |
| Eclipse Temurin JDK | 21.0.13+9 LTS | `~/.local/share/rea/engines/jdk-21.0.13+9` |

`$HOME` is per-chat (`/workspace/homes/<chat-id>`), so these paths are chat-local. Ghidra and the
JDK are ~1.2 GB total; the engine download archives were deleted after extraction.

## Registration

The agent picks REA up from `~/.agents/mcp.json` (FreeBuff's shared MCP registration):

```json
{
  "mcpServers": {
    "rea": {
      "env": {
        "REA_GHIDRA_STARTUP_TIMEOUT_MS": "1800000",
        "GHIDRA_HEADLESS_MAXMEM": "2G",
        "GHIDRA_INSTALL_DIR": "$HOME/.local/share/rea/engines/ghidra_12.1.4_PUBLIC",
        "JAVA_HOME": "$HOME/.local/share/rea/engines/jdk-21.0.13+9"
      },
      "command": "/usr/local/bin/node",
      "args": ["<npx-cache>/node_modules/.bin/rea", "mcp"]
    }
  }
}
```

`env` holds absolute paths in the real file; `$HOME` is abbreviated here for readability.

- `GHIDRA_HEADLESS_MAXMEM=2G` becomes the Ghidra JVM's `-Xmx`. Capped at 2G deliberately: this
  host has ~3.9 GB RAM total, so a larger heap invites the OOM killer. The heap does not bound
  the JVM's RSS.
- `REA_GHIDRA_STARTUP_TIMEOUT_MS=1800000` gives import + auto-analysis + bridge readiness up to
  30 minutes (REA's internal default is 330000).

## Rebuilding after a restore or on a fresh machine

Engines and the JDK live under `$HOME/.local/share/rea/engines` and normally persist with the
workspace. The npx cache **does not** — npm caches are excluded from checkpoints, so after a
restore the registration's `args` path will dangle. Re-run setup to repair it.

```bash
# 1. Engine + JDK (no root required; both extract into the workspace)
mkdir -p "$HOME/.local/share/rea/engines" && cd "$HOME/.local/share/rea/engines"

curl -fL -o jdk21.tar.gz \
  "https://github.com/adoptium/temurin21-binaries/releases/download/jdk-21.0.13%2B9/OpenJDK21U-jdk_x64_linux_hotspot_21.0.13_9.tar.gz"
echo 'd7f5d5cf865a2902b70ff549b97586a7d3f234e016f041ab447ea740c7d66d1c  jdk21.tar.gz' | sha256sum -c -
tar -xzf jdk21.tar.gz

curl -fL -o ghidra.zip \
  "https://github.com/NationalSecurityAgency/ghidra/releases/download/Ghidra_12.1.4_build/ghidra_12.1.4_PUBLIC_20260921.zip"
unzip -tq ghidra.zip && unzip -q ghidra.zip
rm -f jdk21.tar.gz ghidra.zip

# 2. Point REA at them and re-register the client
export JAVA_HOME="$HOME/.local/share/rea/engines/jdk-21.0.13+9"
export GHIDRA_INSTALL_DIR="$HOME/.local/share/rea/engines/ghidra_12.1.4_PUBLIC"
export GHIDRA_HEADLESS_MAXMEM=2G
export REA_GHIDRA_STARTUP_TIMEOUT_MS=1800000

npx -y rea-agents@latest setup --client freebuff --dry-run   # review the plan
npx -y rea-agents@latest setup --client freebuff --yes       # apply registration + skill
```

Ghidra's release page publishes no checksum asset, so only archive integrity (`unzip -t`) can be
checked for it; the JDK digest above is Adoptium's published value.

## Verify

```bash
# registration-scoped: reads ~/.agents/mcp.json, no exports needed
npx -y rea-agents@latest doctor --client freebuff --json   # healthy: true, registration: aligned
npx -y rea-agents@latest doctor --skill --json             # skill state: aligned

# provider-scoped: inspects THIS shell, so export the paths first
export JAVA_HOME="$HOME/.local/share/rea/engines/jdk-21.0.13+9"
export GHIDRA_INSTALL_DIR="$HOME/.local/share/rea/engines/ghidra_12.1.4_PUBLIC"
npx -y rea-agents@latest doctor --provider ghidra --json   # healthy: true
```

`doctor --provider ghidra` checks platform, architecture, Ghidra version, `support/analyzeHeadless`,
the `Ghidra/Features/Decompiler/os/linux_x86_64/decompile` native binary, and that the JDK is a
64-bit full JDK satisfying Ghidra's declared `application.java.min` (21 for 12.1.4).

These two scopes read different places, and that trips people up: `doctor --provider …` reads the
environment of the shell running it, so with no exports it reports `GHIDRA_INSTALL_DIR is not set`
even though the registration is perfectly healthy. `doctor --client freebuff` reads the registration
itself (including its `env` block) and needs nothing exported.

## Caveats

- **Restart required.** MCP servers are loaded when the client starts. After any setup change the
  FreeBuff session must restart before REA's tools are callable.
- **`doctor` does not report heap or timeout.** To confirm the heap reached Java, inspect a running
  Ghidra process: `ps -eo args= | grep -o -- '-Xmx[0-9]*[GgMm]'` should show `-Xmx2G`.
- **Only Ghidra is configured.** Hopper, IDA, JADX and Kuna report `unavailable`; that is
  informational and does not affect `healthy`. The JDK already satisfies JADX's "full JDK 17+"
  requirement, so Android/APK support is just the pinned JAR plus an env var:

  ```bash
  curl -fL -o "$HOME/.local/share/rea/android/jadx-headless-mcp-0.7.1-all.jar" \
    "https://github.com/1013503897/jadx-headless-mcp/releases/download/v0.7.1/jadx-headless-mcp-0.7.1-all.jar"
  # sha256 6e5eacf500b64292bfb73c49797c1958f6ee44646e43e868039ae7feb573ff75
  export REA_JADX_MCP_JAR="$HOME/.local/share/rea/android/jadx-headless-mcp-0.7.1-all.jar"
  ```
- **Large targets are RAM-bound.** 2 CPUs and ~3.9 GB RAM handle small-to-medium binaries well;
  heavily sized 50 MB+ targets may still time out or exhaust memory regardless of these settings.
