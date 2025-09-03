# Wazuh Agent (Windows 11) — Install & Enroll (v4.7.4)

> Run **PowerShell as Administrator**.

---

## 0) Prerequisites

* Decide your **manager address**:

  * **Same PC (Docker & agent on one machine):** `127.0.0.1`
  * **LAN manager:** your host’s LAN IP (e.g., `192.168.1.12`)
* Manager ports to be open on the **manager host**:
  **1514/TCP** (agent), **1515/TCP** (enrollment), **55000/TCP** (API)

---

## 1) Set variables

```powershell
$ManagerIP  = '192.168.1.12'      # or '127.0.0.1' if same PC
$AgentName  = $env:COMPUTERNAME   # or 'CHESTER'
$MsiUrlCDN  = 'https://d325fvi1o3ej6r.cloudfront.net/4.x/windows/wazuh-agent-4.7.4-1.msi'

$MsiPath    = "$env:TEMP\wazuh-agent-4.7.4-1.msi"
$InstallDir = 'C:\Program Files (x86)\ossec-agent'
$MsiLog     = "$env:TEMP\wazuh-install.log"
```

---

## 2) Download the MSI

```powershell
curl.exe -L $MsiUrlCDN -o $MsiPath
Get-Item $MsiPath | Format-List Name,Length,FullName
```

> If the download has DNS/proxy issues, try your browser with the same URL and save to `Downloads`, then adjust `$MsiPath`.

---

## 3) Silent install (with full logging)

```powershell
$exit = (Start-Process msiexec.exe -PassThru -Wait -Verb RunAs -ArgumentList @(
  "/i `"$MsiPath`"",
  "/qn",
  "ALLUSERS=1",
  "WAZUH_MANAGER=$ManagerIP",
  "WAZUH_REGISTRATION_SERVER=$ManagerIP",
  "WAZUH_AGENT_NAME=$AgentName",
  # Optional: add if your manager requires enrollment password:
  # "WAZUH_REGISTRATION_PASSWORD=YourStrongPassword",
  "/l*v `"$MsiLog`""
)).ExitCode

"MSI ExitCode: $exit"  # 0=success, 3010=reboot required
```

If you get **3010**, reboot once, then continue.

---

## 4) Register/start the Windows service

```powershell
# Service may be named Wazuh (service name WazuhSvc)
$svc = Get-Service -ErrorAction SilentlyContinue | Where-Object { $_.Name -match '^Wazuh' -or $_.DisplayName -match '^Wazuh' }
if (-not $svc) {
  & "$InstallDir\wazuh-agent.exe" install-service
  Start-Sleep 2
  $svc = Get-Service | Where-Object { $_.Name -match '^Wazuh' }
}

Start-Service -Name $svc.Name
Get-Service -Name $svc.Name
```

---

## 5) Point the agent to the **correct** manager address

```powershell
notepad "$InstallDir\ossec.conf"
```

Ensure this block matches your scenario:

```xml
<client>
  <server>
    <address>127.0.0.1</address>   <!-- or 192.168.1.12 for LAN -->
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

Save the file.

---

## 6) Enroll the agent (obtain key)

**Same PC (Docker & agent):**

```powershell
& "$InstallDir\agent-auth.exe" -m 127.0.0.1 -p 1515
```

**OR — LAN manager:**

```powershell
& "$InstallDir\agent-auth.exe" -m 192.168.1.12 -p 1515
```

(Optionally set a specific display name)

```powershell
& "$InstallDir\agent-auth.exe" -m 127.0.0.1 -p 1515 -A chester-win11
```

> If you see **“Duplicate agent name”**, remove the old record from the manager and delete `client.keys`, then re-run `agent-auth.exe`.
> (Manager removal: `docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/manage_agents -l` → note ID → `-r <ID>`.)

---

## 7) Restart & verify

```powershell
Restart-Service Wazuh
Get-Content "$InstallDir\ossec.log" -Tail 120
```

**Good signs in the log:**

* `Using server address: ...`
* `Connected to server`
* `Agent is now connected`

Also check Wazuh Dashboard → **Agents** for your host status = **connected**.

---

## 8) Connectivity quick tests (from the agent PC)

```powershell
# Use the same address you used above
Test-NetConnection $ManagerIP -Port 1515
Test-NetConnection $ManagerIP -Port 1514
Test-NetConnection $ManagerIP -Port 55000
```

All should say `TcpTestSucceeded : True`.

---

## 9) Manager host firewall (if Windows host)

Run these on the **manager** Windows host (once):

```powershell
New-NetFirewallRule -DisplayName "Wazuh 1514 TCP" -Direction Inbound -Protocol TCP -LocalPort 1514 -Action Allow
New-NetFirewallRule -DisplayName "Wazuh 1515 TCP" -Direction Inbound -Protocol TCP -LocalPort 1515 -Action Allow
New-NetFirewallRule -DisplayName "Wazuh 55000 TCP" -Direction Inbound -Protocol TCP -LocalPort 55000 -Action Allow
```

Confirm Docker published the ports:

```powershell
docker ps --format "table {{.Names}}`t{{.Ports}}"
# Expect: 0.0.0.0:1514-1515->1514-1515/tcp and 0.0.0.0:55000->55000/tcp
```

---

## One-liner install (fast path)

```powershell
$ManagerIP='192.168.1.12'; $AgentName=$env:COMPUTERNAME; $u='https://d325fvi1o3ej6r.cloudfront.net/4.x/windows/wazuh-agent-4.7.4-1.msi'; $m="$env:TEMP\wazuh-agent-4.7.4-1.msi"; curl.exe -L $u -o $m; Start-Process msiexec -ArgumentList "/i `"$m`" /qn ALLUSERS=1 WAZUH_MANAGER=$ManagerIP WAZUH_REGISTRATION_SERVER=$ManagerIP WAZUH_AGENT_NAME=$AgentName" -Wait -Verb RunAs; Start-Service Wazuh
```

---

## Troubleshooting

* **“Unable to connect to enrollment service”**: use the correct manager address (127.0.0.1 if same PC, or the right LAN IP), ensure routing/subnet is correct, and verify ports 1514/1515/55000.
* **Duplicate agent name**: delete the old record on the manager and remove `client.keys`, then re-enroll.
* **Service missing after install**: run `"$InstallDir\wazuh-agent.exe" install-service`, then `Start-Service Wazuh`.
* **MSI exit code 3010**: reboot, then continue.
* **DNS issues downloading**: use the CloudFront URL above or download with a browser.

---
