# opencode-mobile

Use this skill when a user wants to run OpenCode, Codex, or Cursor from a phone without requiring the phone and computer to be on the same Wi-Fi network.

## What this skill provides

**opencode-mobile** is a practical Windows setup for using the Clawdex Mobile iOS/Android app as a mobile client for OpenCode and similar coding agents.

It is designed for the real-world mobile case:

- phone is on 4G/5G, office Wi-Fi, hotel Wi-Fi, campus Wi-Fi, or any other network
- computer is behind NAT or a corporate/router firewall
- Tailscale or ZeroTier is unavailable on the phone
- direct LAN access is unreliable because public Wi-Fi often blocks device-to-device traffic
- opencode web works but feels slow or poor on mobile

The setup uses:

- `clawdex-mobile` as the bridge runtime
- the Clawdex Mobile app as the phone UI
- `cloudflared` quick tunnels for no-domain, no-router-config remote access
- a hardened local bridge that binds to `127.0.0.1` only
- QR pairing so the user does not manually copy the changing `trycloudflare.com` URL

## Security model

Default remote mode is intentionally conservative:

- The bridge listens on `127.0.0.1:8787`, not on the LAN.
- The public endpoint is created by `cloudflared` as an outbound HTTPS tunnel.
- No router port forwarding is required.
- `BRIDGE_ALLOW_QUERY_TOKEN_AUTH=false` is enforced, so the token is not accepted from URL query strings.
- The Clawdex app pairs by scanning a QR containing `bridgeUrl` and `bridgeToken`.
- The Cloudflare quick tunnel URL is temporary and changes after restart.

Do not publish `.env.secure`, bridge logs, tunnel logs, screenshots containing QR codes, or bridge tokens.

## Use cases

### Remote mobile mode: recommended default

Use this when the phone and computer are not on the same network, or when the network blocks peer-to-peer LAN traffic.

Examples:

- company Wi-Fi
- campus Wi-Fi
- hotel Wi-Fi
- airport Wi-Fi
- phone on cellular data
- computer at home, phone outside

The user runs one PowerShell script. It starts the bridge, creates a Cloudflare quick tunnel, patches the bridge connect URL, and prints a QR code in the terminal. The phone scans that QR in the Clawdex Mobile app.

### LAN mode: optional fast path

Use this only when phone and computer are on a trusted same Wi-Fi or hotspot and the network allows device-to-device traffic.

This avoids Cloudflare and is faster, but exposes the bridge on the LAN. It is not the default.

## Phone setup

Install **Clawdex Mobile** on the phone.

Open the app and choose the onboarding / add bridge / scan QR flow. In remote mode, scan the QR printed by `remote.ps1`.

If the app has an old bridge entry, delete it and pair again. Old entries may point to a stale LAN IP or stale quick-tunnel URL.

## Computer prerequisites

Windows PowerShell 5.1 or later.

Install Node.js and npm first.

Then install:

```powershell
npm install -g opencode-ai
npm install -g clawdex-mobile@latest
winget install --id Cloudflare.cloudflared --accept-package-agreements --accept-source-agreements
```

Git Bash is required because the current clawdex setup helper is a bash script:

```powershell
winget install --id Git.Git --accept-package-agreements --accept-source-agreements
```

## One-time local folder

Create a local working folder, for example:

```powershell
New-Item -ItemType Directory -Force C:\clawdex | Out-Null
```

Copy `remote.ps1` from this skill into `C:\clawdex\remote.ps1`.

Optionally copy `lan.ps1` into `C:\clawdex\lan.ps1` if LAN mode is needed.

## Remote mode script

Save this as `C:\clawdex\remote.ps1`.

```powershell
param(
  [int]$Port = 8787,
  [string]$Root = "C:\clawdex"
)

$ErrorActionPreference = "Stop"

function Resolve-RequiredCommandPath($Name) {
  $cmd = Get-Command $Name -ErrorAction SilentlyContinue
  if (-not $cmd) { throw "Required command not found: $Name" }
  return $cmd.Source
}

function Resolve-NpmGlobalRoot {
  $root = (& npm root -g).Trim()
  if (-not $root) { throw "Unable to resolve npm global root." }
  return $root
}

function Resolve-CloudflaredPath($Root) {
  $candidates = @(
    (Join-Path $Root "cloudflared.exe"),
    (Get-Command cloudflared.exe -ErrorAction SilentlyContinue).Source,
    (Get-Command cloudflared -ErrorAction SilentlyContinue).Source
  ) | Where-Object { $_ }

  foreach ($candidate in $candidates) {
    if (Test-Path -LiteralPath $candidate) { return $candidate }
  }
  throw "cloudflared not found. Install with: winget install --id Cloudflare.cloudflared"
}

function Resolve-LanIP {
  $ip = (Get-NetIPAddress -AddressFamily IPv4 |
    Where-Object {
      $_.IPAddress -notlike "127.*" -and
      $_.IPAddress -notlike "169.*" -and
      $_.PrefixOrigin -ne "WellKnown"
    } |
    Select-Object -First 1).IPAddress

  if (-not $ip) { throw "No LAN IPv4 address found." }
  return $ip
}

function Stop-ListeningProcesses($Ports) {
  $pids = Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue |
    Where-Object { $_.LocalPort -in $Ports } |
    Select-Object -ExpandProperty OwningProcess -Unique

  foreach ($pid in $pids) {
    try { Stop-Process -Id $pid -Force -ErrorAction Stop } catch { }
  }
}

New-Item -ItemType Directory -Force $Root | Out-Null
$LogDir = Join-Path $Root "logs"
New-Item -ItemType Directory -Force $LogDir | Out-Null

$NpmRoot = Resolve-NpmGlobalRoot
$ClawdexPkg = Join-Path $NpmRoot "clawdex-mobile"
if (-not (Test-Path -LiteralPath $ClawdexPkg)) {
  throw "clawdex-mobile not found. Install with: npm install -g clawdex-mobile@latest"
}

$GitBash = Resolve-RequiredCommandPath "bash.exe"
$OpencodeCmd = Resolve-RequiredCommandPath "opencode.cmd"
$Cloudflared = Resolve-CloudflaredPath $Root
$EnvFile = Join-Path $Root ".env.secure"
$CfLog = Join-Path $LogDir "cloudflared.log"
$CfErrLog = Join-Path $LogDir "cloudflared.err.log"

Write-Host "Preparing opencode-mobile remote bridge..." -ForegroundColor Cyan
Stop-ListeningProcesses @($Port, ($Port + 1), 4090)
Get-Process cloudflared -ErrorAction SilentlyContinue | Stop-Process -Force -ErrorAction SilentlyContinue
Start-Sleep -Seconds 1

$SetupHost = Resolve-LanIP
$env:CLAWDEX_WORKSPACE_ROOT = $Root
$env:BRIDGE_NETWORK_MODE = "local"
$env:BRIDGE_HOST_OVERRIDE = $SetupHost
$env:BRIDGE_ACTIVE_ENGINE = "opencode"
$env:BRIDGE_ENABLED_ENGINES = "opencode"

& $GitBash (Join-Path $ClawdexPkg "scripts\setup-secure-dev.sh")
if ($LASTEXITCODE -ne 0) { throw "clawdex secure setup failed." }

Write-Host "Starting Cloudflare quick tunnel..." -ForegroundColor Cyan
"" | Out-File -FilePath $CfLog -Encoding ASCII -Force
"" | Out-File -FilePath $CfErrLog -Encoding ASCII -Force
$cf = Start-Process -FilePath $Cloudflared `
  -ArgumentList @("tunnel", "--url", "http://127.0.0.1:$Port", "--no-autoupdate") `
  -RedirectStandardOutput $CfLog `
  -RedirectStandardError $CfErrLog `
  -PassThru `
  -WindowStyle Hidden

$publicUrl = $null
$deadline = (Get-Date).AddSeconds(45)
while ((Get-Date) -lt $deadline) {
  Start-Sleep -Milliseconds 500
  $combined = @()
  if (Test-Path $CfLog) { $combined += Get-Content $CfLog }
  if (Test-Path $CfErrLog) { $combined += Get-Content $CfErrLog }
  $hit = $combined | Select-String -Pattern 'https://[a-z0-9-]+\.trycloudflare\.com' -AllMatches | Select-Object -First 1
  if ($hit) {
    $publicUrl = $hit.Matches[0].Value
    break
  }
  if ($cf.HasExited) { throw "cloudflared exited unexpectedly. Check $CfLog and $CfErrLog" }
}
if (-not $publicUrl) {
  try { Stop-Process -Id $cf.Id -Force -ErrorAction Stop } catch { }
  throw "Timed out waiting for Cloudflare quick tunnel URL. Check $CfLog and $CfErrLog"
}

$lines = Get-Content -LiteralPath $EnvFile
$lines = $lines -replace '^BRIDGE_HOST=.*$', 'BRIDGE_HOST=127.0.0.1'
$lines = $lines -replace '^BRIDGE_CONNECT_URL=.*$', "BRIDGE_CONNECT_URL=$publicUrl"
$lines = $lines -replace '^BRIDGE_ALLOW_QUERY_TOKEN_AUTH=.*$', 'BRIDGE_ALLOW_QUERY_TOKEN_AUTH=false'
$lines = $lines -replace '^OPENCODE_CLI_BIN=.*$', "OPENCODE_CLI_BIN=$($OpencodeCmd -replace '\\', '/')"
$lines | Set-Content -LiteralPath $EnvFile -Encoding ASCII

$global:__opencodeMobileCloudflaredPid = $cf.Id
$cleanup = {
  Write-Host ""
  Write-Host "Stopping opencode-mobile bridge..." -ForegroundColor Yellow
  try { Stop-Process -Id $global:__opencodeMobileCloudflaredPid -Force -ErrorAction Stop } catch { }
  $pids = Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue |
    Where-Object { $_.LocalPort -in 8787, 8788, 4090 } |
    Select-Object -ExpandProperty OwningProcess -Unique
  foreach ($pid in $pids) {
    try { Stop-Process -Id $pid -Force -ErrorAction Stop } catch { }
  }
}
Register-EngineEvent -SourceIdentifier PowerShell.Exiting -Action $cleanup | Out-Null

Write-Host ""
Write-Host "Remote URL: $publicUrl" -ForegroundColor Green
Write-Host "Starting bridge. Scan the QR printed below in Clawdex Mobile." -ForegroundColor Yellow
Write-Host "Press Ctrl+C to stop." -ForegroundColor Yellow
Write-Host ""

$env:CLAWDEX_WORKSPACE_ROOT = $Root
try {
  & node (Join-Path $ClawdexPkg "scripts\start-bridge-secure.js")
} finally {
  & $cleanup
}
```

## Optional LAN script

Save this as `C:\clawdex\lan.ps1` only if same-Wi-Fi direct mode is useful.

```powershell
param(
  [int]$Port = 8787,
  [string]$Root = "C:\clawdex"
)

$ErrorActionPreference = "Stop"

function Resolve-RequiredCommandPath($Name) {
  $cmd = Get-Command $Name -ErrorAction SilentlyContinue
  if (-not $cmd) { throw "Required command not found: $Name" }
  return $cmd.Source
}

function Resolve-NpmGlobalRoot {
  $root = (& npm root -g).Trim()
  if (-not $root) { throw "Unable to resolve npm global root." }
  return $root
}

function Resolve-LanIP {
  $ip = (Get-NetIPAddress -AddressFamily IPv4 |
    Where-Object {
      $_.IPAddress -notlike "127.*" -and
      $_.IPAddress -notlike "169.*" -and
      $_.PrefixOrigin -ne "WellKnown"
    } |
    Select-Object -First 1).IPAddress
  if (-not $ip) { throw "No LAN IPv4 address found." }
  return $ip
}

New-Item -ItemType Directory -Force $Root | Out-Null
$NpmRoot = Resolve-NpmGlobalRoot
$ClawdexPkg = Join-Path $NpmRoot "clawdex-mobile"
$GitBash = Resolve-RequiredCommandPath "bash.exe"
$OpencodeCmd = Resolve-RequiredCommandPath "opencode.cmd"
$IP = Resolve-LanIP

$pids = Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue |
  Where-Object { $_.LocalPort -in $Port, ($Port + 1), 4090 } |
  Select-Object -ExpandProperty OwningProcess -Unique
foreach ($pid in $pids) {
  try { Stop-Process -Id $pid -Force -ErrorAction Stop } catch { }
}

$env:CLAWDEX_WORKSPACE_ROOT = $Root
$env:BRIDGE_NETWORK_MODE = "local"
$env:BRIDGE_HOST_OVERRIDE = $IP
$env:BRIDGE_ACTIVE_ENGINE = "opencode"
$env:BRIDGE_ENABLED_ENGINES = "opencode"

& $GitBash (Join-Path $ClawdexPkg "scripts\setup-secure-dev.sh")
if ($LASTEXITCODE -ne 0) { throw "clawdex secure setup failed." }

$EnvFile = Join-Path $Root ".env.secure"
$lines = Get-Content -LiteralPath $EnvFile
$lines = $lines -replace '^BRIDGE_ALLOW_QUERY_TOKEN_AUTH=.*$', 'BRIDGE_ALLOW_QUERY_TOKEN_AUTH=false'
$lines = $lines -replace '^OPENCODE_CLI_BIN=.*$', "OPENCODE_CLI_BIN=$($OpencodeCmd -replace '\\', '/')"
$lines | Set-Content -LiteralPath $EnvFile -Encoding ASCII

Write-Host "LAN URL: http://$IP`:$Port" -ForegroundColor Green
Write-Host "Starting bridge. Scan the QR printed below in Clawdex Mobile." -ForegroundColor Yellow
& node (Join-Path $ClawdexPkg "scripts\start-bridge-secure.js")
```

## Daily usage

Remote mode:

```powershell
C:\clawdex\remote.ps1
```

Then scan the QR printed by the bridge.

LAN mode:

```powershell
C:\clawdex\lan.ps1
```

Then scan the QR printed by the bridge.

## Troubleshooting

### `failed to run setup-wizard.sh UNKNOWN`

This is expected on Windows when running `clawdex init` directly. The npm CLI tries to spawn a `.sh` script. The scripts in this skill call the underlying shell helper through Git Bash instead.

### `failed to start opencode serve: program not found`

The bridge tried to run `opencode` instead of the Windows `opencode.cmd` shim. The scripts patch `OPENCODE_CLI_BIN` to the full `opencode.cmd` path.

### Phone says connection error on same Wi-Fi

Many company, campus, hotel, and public Wi-Fi networks block device-to-device traffic. Use remote mode instead of LAN mode.

### Cloudflare tunnel starts, but mobile cannot connect

Check the terminal. If the quick tunnel URL changed, delete the old bridge entry in Clawdex Mobile and scan the new QR.

### It is slow

Cloudflare quick tunnels may route through distant edge locations. The benefit is network freedom, not guaranteed low latency. LAN mode is faster when available.

## Publishing checklist

Before publishing or sharing logs:

- Do not commit `C:\clawdex\.env.secure`.
- Do not commit logs from `C:\clawdex\logs`.
- Do not share screenshots of the pairing QR.
- Do not share `BRIDGE_AUTH_TOKEN`.
- Keep `BRIDGE_ALLOW_QUERY_TOKEN_AUTH=false` for remote mode.
