# Wazuh Agent (Windows 11) — Stop, Unregister & Remove

> Run **PowerShell as Administrator**.
> This removes the agent from your Windows machine **and** deletes its record on the Wazuh manager.


## 0) Quick context

* **Windows agent install dir (default):** `C:\Program Files (x86)\ossec-agent`
* **Manager container (your setup):** `single-node-wazuh.manager-1`
* **Ports on manager:** 1514/TCP (agent), 1515/TCP (enrollment), 55000/TCP (API)


## 1) Stop the agent service (Windows)

```powershell
# Stop (either name, depending on build)
Stop-Service Wazuh    -ErrorAction SilentlyContinue
Stop-Service WazuhSvc -ErrorAction SilentlyContinue
```

---

## 2) Uninstall the Windows agent

### Option A — You still have the MSI

```powershell
$msi = "$env:TEMP\wazuh-agent-4.7.4-1.msi"
Start-Process msiexec.exe -Wait -Verb RunAs -ArgumentList "/x `"$msi`" /qn /l*v `"$env:TEMP\wazuh-uninstall.log`""
```

### Option B — Use the uninstall entry from the registry

```powershell
$apps = @(
  "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall",
  "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall"
)

$uninst = Get-ChildItem $apps -ErrorAction SilentlyContinue |
  ForEach-Object { Get-ItemProperty $_.PSPath } |
  Where-Object { $_.DisplayName -like "Wazuh Agent*" } |
  Select-Object -First 1 -ExpandProperty UninstallString

if ($uninst) {
  $cmd = $uninst -replace '/I','/X' -replace '/i','/x'
  Start-Process msiexec.exe -Wait -Verb RunAs -ArgumentList "$cmd /qn /l*v `"$env:TEMP\wazuh-uninstall.log`""
} else {
  Write-Host "Wazuh Agent uninstall entry not found."
}
```

**(Optional)** Clear the local enrollment key before/after uninstall:

```powershell
$dir = "C:\Program Files (x86)\ossec-agent"
If (Test-Path "$dir\client.keys") { Remove-Item "$dir\client.keys" -Force }
```

**Cleanup leftovers & stale service entry:**

```powershell
Remove-Item -Recurse -Force "C:\Program Files (x86)\ossec-agent" -ErrorAction SilentlyContinue
sc.exe delete WazuhSvc | Out-Null
```

---

## 3) Remove the agent from the **manager** (Docker)

### A) List → remove by **ID** (recommended)

```powershell
# List all agents and note the ID you want to remove
docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/manage_agents -l

# Remove by ID (replace 006 with the ID you saw)
docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/manage_agents -r 006

# Verify it’s gone
docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/manage_agents -l
```

### B) Remove by **name** (e.g., “chester”)

```powershell
$mgr = 'single-node-wazuh.manager-1'
$name = 'chester'

# Grab IDs whose row contains the name
$ids = docker exec $mgr /var/ossec/bin/manage_agents -l |
  Select-String -Pattern $name |
  ForEach-Object { ($_ -split '\s+')[0] }

# Remove each matching ID
$ids | ForEach-Object {
  docker exec -it $mgr /var/ossec/bin/manage_agents -r $_
}

# Re-list to confirm
docker exec -it $mgr /var/ossec/bin/manage_agents -l
```

### C) Dashboard UI

* **Agents** → find your host → **⋯ → Remove** → confirm.

> Prefer `docker compose`? In the correct compose directory:
>
> ```powershell
> docker compose exec wazuh.manager /var/ossec/bin/manage_agents -l
> docker compose exec wazuh.manager /var/ossec/bin/manage_agents -r 006
> ```

---

## 4) Verify cleanup

**On Windows:**

```powershell
# No Wazuh services
Get-Service | Where-Object { $_.Name -match 'Wazuh' -or $_.DisplayName -match 'Wazuh' }

# No agent folder
Test-Path "C:\Program Files (x86)\ossec-agent"   # should be False
```

**On manager:**

* `manage_agents -l` no longer lists the agent.
* Dashboard → **Agents** no longer shows the host.

---

## Troubleshooting

* **Agent still shows in Dashboard:** Remove it on the manager (`manage_agents -r <ID>` or via UI) — uninstalling on Windows alone doesn’t delete the manager record.
* **“service wazuh.manager is not running”:** You’re in a different compose project/folder. Use the container name (`single-node-wazuh.manager-1`) or run `docker ps` to find the right one.
* **Can’t find container name?**

  ```powershell
  docker ps --format "table {{.Names}}`t{{.Image}}`t{{.Status}}`t{{.Ports}}"
  ```

---

### One-shot PowerShell (Windows cleanup)

```powershell
# Stop & remove Windows agent completely
Stop-Service Wazuh    -ErrorAction SilentlyContinue
Stop-Service WazuhSvc -ErrorAction SilentlyContinue

$dir = "C:\Program Files (x86)\ossec-agent"
If (Test-Path "$dir\client.keys") { Remove-Item "$dir\client.keys" -Force }

$apps = @(
  "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall",
  "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall"
)
$uninst = Get-ChildItem $apps -ea 0 | % { Get-ItemProperty $_.PSPath } |
  ? { $_.DisplayName -like "Wazuh Agent*" } |
  Select-Object -First 1 -ExpandProperty UninstallString

if ($uninst) {
  $cmd = $uninst -replace '/I','/X' -replace '/i','/x'
  Start-Process msiexec.exe -Wait -Verb RunAs -ArgumentList "$cmd /qn /l*v `"$env:TEMP\wazuh-uninstall.log`""
}

Remove-Item -Recurse -Force $dir -ErrorAction SilentlyContinue
sc.exe delete WazuhSvc | Out-Null
```

 
