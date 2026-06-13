# Agent Harness Usage: Claude Code, Codex/LazyCodex, and GJC

This page records a Windows operator setup for using Headroom alongside three active coding-agent workflows:

- Claude Code
- Codex with LazyCodex
- Gajae-Code (`gjc`)

The goal is to keep existing sessions and project-specific agent configuration safe while allowing new sessions to opt in to Headroom's local proxy compression.

## Recommended operating model

Run one local Headroom proxy and attach only the new agent sessions that should use it:

```text
Headroom proxy on 127.0.0.1:8787
  ├─ Claude Code session launched through Headroom
  ├─ Codex/LazyCodex session launched through Headroom
  └─ GJC session launched with provider base URL environment variables
```

Existing Claude Code, Codex, LazyCodex, or GJC sessions are not affected by this model. Environment variables and wrapper launch state apply only to newly started processes.

## Windows autostart pattern

For an always-on local proxy, register a user logon task that starts the proxy in the background.

Example launcher script:

```powershell
# %USERPROFILE%\.headroom\start-headroom-proxy.ps1
$ErrorActionPreference = 'SilentlyContinue'

$HealthUrl = 'http://127.0.0.1:8787/health'
try {
    $response = Invoke-WebRequest -Uri $HealthUrl -UseBasicParsing -TimeoutSec 2
    if ($response.StatusCode -eq 200) {
        exit 0
    }
} catch {
    # Proxy is not running yet.
}

$env:HEADROOM_TELEMETRY = 'off'
$env:PYTHONIOENCODING = 'utf-8'

$HeadroomExe = Join-Path $env:USERPROFILE '.venvs\headroom\Scripts\headroom.exe'
$LogDir = Join-Path $env:USERPROFILE '.headroom\logs'
New-Item -ItemType Directory -Force -Path $LogDir | Out-Null

$StdoutLog = Join-Path $LogDir 'headroom-proxy.out.log'
$StderrLog = Join-Path $LogDir 'headroom-proxy.err.log'

Start-Process -FilePath $HeadroomExe `
    -ArgumentList @('proxy', '--port', '8787', '--no-telemetry') `
    -WindowStyle Hidden `
    -RedirectStandardOutput $StdoutLog `
    -RedirectStandardError $StderrLog
```

Register it with Task Scheduler:

```powershell
$Action = New-ScheduledTaskAction `
    -Execute 'powershell.exe' `
    -Argument '-NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File "%USERPROFILE%\.headroom\start-headroom-proxy.ps1"'
$Trigger = New-ScheduledTaskTrigger -AtLogOn
$Settings = New-ScheduledTaskSettingsSet `
    -AllowStartIfOnBatteries `
    -DontStopIfGoingOnBatteries `
    -MultipleInstances IgnoreNew `
    -RestartCount 3 `
    -RestartInterval (New-TimeSpan -Minutes 1)
Register-ScheduledTask `
    -TaskName 'Headroom Proxy Autostart' `
    -Action $Action `
    -Trigger $Trigger `
    -Settings $Settings `
    -Description 'Start Headroom proxy on user logon.' `
    -Force
```

Verify:

```powershell
Invoke-RestMethod http://127.0.0.1:8787/health
```

Disable autostart:

```powershell
Unregister-ScheduledTask -TaskName 'Headroom Proxy Autostart' -Confirm:$false
```

## Claude Code

Use a wrapper command that reuses the persistent proxy and avoids changing Claude's MCP or context-tool configuration:

```cmd
@echo off
headroom wrap claude --no-proxy --no-context-tool --no-mcp --no-serena %*
```

Example name:

```text
claude-headroom.cmd
```

Run:

```powershell
claude-headroom
```

This applies Headroom only to the launched Claude Code process. Existing Claude Code sessions remain unchanged.

## Codex with LazyCodex

LazyCodex manages Codex skills, hooks, agents, and configuration. The safest Headroom integration is to route model traffic through the proxy but skip extra Headroom context/MCP/Serena setup:

```cmd
@echo off
headroom wrap codex --no-proxy --no-context-tool --no-mcp --no-serena %*
```

Example name:

```text
codex-headroom.cmd
```

Run:

```powershell
codex-headroom
codex-headroom -- "fix the bug"
```

Inside that Codex session, LazyCodex skills remain available. Use the normal LazyCodex `$` commands, for example:

```text
$init-deep
$ulw-plan "task description"
$start-work
$ulw-loop "task description"
```

### Codex config mutation and cleanup

`headroom wrap codex` injects a Headroom provider block into the active Codex `config.toml` so both HTTP and WebSocket Codex traffic can route through the proxy. This is intentional, but it means a later plain `codex` launch can continue using Headroom until the block is removed.

Recommended cleanup command:

```cmd
@echo off
headroom unwrap codex --no-stop-proxy %*
```

Example name:

```text
codex-headroom-off.cmd
```

Run after finishing a Headroom-routed Codex/LazyCodex session:

```powershell
codex-headroom-off
```

## GJC

GJC is not a direct `headroom wrap` target. It can still use Headroom through provider base URL environment variables. GJC's model registry supports provider base URL overrides for the built-in provider names:

- `ANTHROPIC_BASE_URL` for Anthropic/Claude models
- `OPENAI_BASE_URL` for OpenAI/Codex/GPT models

Because GJC can choose models internally across roles such as default, smol, slow, and plan, a combined launcher is the safest default:

```cmd
@echo off
set "ANTHROPIC_BASE_URL=http://127.0.0.1:8787"
set "OPENAI_BASE_URL=http://127.0.0.1:8787/v1"
gjc %*
```

Example name:

```text
gjc-headroom.cmd
```

Run:

```powershell
gjc-headroom --tmux
gjc-headroom --tmux --model gpt-5.5
gjc-headroom --tmux --model opus
```

Provider-specific launchers are also useful when the operator wants stricter routing:

```cmd
:: gjc-headroom-openai.cmd
@echo off
set "OPENAI_BASE_URL=http://127.0.0.1:8787/v1"
gjc %*
```

```cmd
:: gjc-headroom-claude.cmd
@echo off
set "ANTHROPIC_BASE_URL=http://127.0.0.1:8787"
gjc %*
```

## What `--no-context-tool` means

`--no-context-tool` is the same safety intent as `--no-rtk`: skip setup of Headroom's CLI context-tool helper.

Without this flag, Headroom may install or configure helper tooling such as RTK or lean-ctx and inject instructions into agent guidance files, for example Codex `AGENTS.md`. That can be useful in a clean environment, but it is risky when the operator already has active Claude Code customization, LazyCodex hooks, GJC workflows, and multiple in-progress projects.

Use this conservative default in mixed-agent environments:

```text
--no-context-tool --no-mcp --no-serena
```

This keeps Headroom focused on proxy routing and compression while avoiding extra agent-instruction or MCP changes.

## Safety checklist

Before starting a Headroom-routed session:

1. Confirm the proxy is healthy:

   ```powershell
   Invoke-RestMethod http://127.0.0.1:8787/health
   ```

2. Use an explicit launcher instead of replacing the base commands:

   ```powershell
   claude-headroom
   codex-headroom
   gjc-headroom --tmux
   ```

3. After Codex/LazyCodex work, remove Headroom's Codex provider block if plain Codex should return to direct routing:

   ```powershell
   codex-headroom-off
   ```

4. Keep plain commands available for non-Headroom sessions:

   ```powershell
   claude
   codex
   gjc --tmux
   ```
