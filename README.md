# Wazuh Docker (Windows) — Replace **v4.7.4** with a New Version (C:\ **or** D:)

> You can install Wazuh on **either your primary drive `C:\` or another drive like `D:\`**.
> In our demo we used **`D:\`**. In the commands below, just set the `DRIVE` variable to `C:` if you prefer your system drive.

```powershell
# ----- Choose where to install -----
$DRIVE   = "D:"      # change to "C:" if you prefer
$NEW_VER = "vX.Y.Z"  # e.g. "v4.9.3" or "v4.12.0"
$BASE    = "$DRIVE\wazuh-$NEW_VER"
```

---

## 1) Stop & remove the current Wazuh (v4.7.4) stack

If you’re inside the compose folder you originally used for 4.7.4:

```powershell
docker compose down
```

If you don’t have that folder handy, remove by image/ancestor (targets **4.7.4**):

```powershell
docker ps -a -q --filter ancestor=wazuh/wazuh-manager:4.7.4   | % { docker rm -f $_ }
docker ps -a -q --filter ancestor=wazuh/wazuh-indexer:4.7.4   | % { docker rm -f $_ }
docker ps -a -q --filter ancestor=wazuh/wazuh-dashboard:4.7.4 | % { docker rm -f $_ }
```

**Free common ports (1514/1515) if still busy:**

```powershell
docker ps --format "table {{.Names}}\t{{.Ports}}" | findstr 1514
# docker stop <name>; docker rm <name>
```

**If a Windows process owns a port (not a container):**

```powershell
Get-NetTCPConnection -LocalPort 1514 -ErrorAction SilentlyContinue | Select LocalAddress,LocalPort,State,OwningProcess
Get-Process -Id <PID>
# Stop-Process -Id <PID> -Force  # if safe/necessary
```

---

## 2) (Optional) Remove old Wazuh volumes for a clean slate

> ⚠️ Destructive — erases saved settings/data.

```powershell
docker volume ls -q --filter "name=wazuh" | % { docker volume rm $_ }
```

---

## 3) (Optional) Remove old **v4.7.4** images to free disk

```powershell
docker rmi wazuh/wazuh-manager:4.7.4 `
          wazuh/wazuh-indexer:4.7.4 `
          wazuh/wazuh-dashboard:4.7.4
# Optional (will re-pull when needed)
# docker rmi wazuh/wazuh-certs-generator:0.0.2
```

---

## 4) Install your **new** Wazuh version on \**C:\ or D:\**

Create the working folder and clone the exact tag:

```powershell
mkdir $BASE
cd    $BASE

git clone https://github.com/wazuh/wazuh-docker.git -b $NEW_VER
cd .\wazuh-docker\single-node
```

### 4.1) Generate TLS certificates

Make sure `config\certs.yml` is a **file** (not a folder):

```powershell
Get-Item .\config\certs.yml | Format-List Mode,Name,Length,FullName
# If Mode shows 'd----', fix it:
# Remove-Item -Recurse -Force .\config\certs.yml
# git checkout -- .\config\certs.yml
```

Generate the certs:

```powershell
docker compose -f .\generate-indexer-certs.yml run --rm generator
# Quick check:
Get-ChildItem .\config\wazuh_indexer_ssl_certs -Recurse
```

### 4.2) (Optional) Use a unique project name

```powershell
$env:COMPOSE_PROJECT_NAME = "wazuh-$NEW_VER"
```

### 4.3) (Optional) Preempt port conflicts by remapping

Edit `docker-compose.yml` (service `wazuh.manager`) if 1514/1515 may be busy:

```yaml
ports:
  - "15140:1514"
  - "15140:1514/udp"
  - "15150:1515"
  - "5140:514/udp"
  - "55000:55000"
# Dashboard examples if needed:
#  - "8443:443"
#  - "56010:5601"
```

### 4.4) Start the new stack

```powershell
docker compose up -d
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

> First boot may take a bit while the indexer initializes; transient “not ready yet” messages are normal.

### 4.5) Open the dashboard

Find the mapped dashboard port:

```powershell
docker ps --format "table {{.Names}}\t{{.Ports}}" | findstr dashboard
```

* `0.0.0.0:5601->5601/tcp` → **[http://localhost:5601](http://localhost:5601)**
* `0.0.0.0:443->443/tcp`   → **[https://localhost](https://localhost)** (accept the self-signed cert)

Log in with the default credentials for that version, then change the password.

---

## 5) Troubleshooting quickies

**Port already allocated (1514/1515):**

```powershell
docker ps --format "table {{.Names}}\t{{.Ports}}" | findstr 1514
# stop/remove the offender, OR remap ports in docker-compose.yml (see 4.3), then:
docker compose up -d
```

**Cert generator error mentioning `/config/certs.yml`:**

* Ensure `.\config\certs.yml` is a **file** and exists, then re-run:

```powershell
docker compose -f .\generate-indexer-certs.yml run --rm generator
```

**“Dashboard not ready yet” at first start:**

```powershell
docker compose logs -f wazuh.indexer
docker compose logs -f wazuh.dashboard
```

---

## 6) Reset sequence (if you want to start over)

```powershell
docker compose down -v
Remove-Item -Recurse -Force .\config\wazuh_indexer_ssl_certs
docker compose -f .\generate-indexer-certs.yml run --rm generator
docker compose up -d
```

---

### ✅ Note

* Old **v4.7.4** stack stopped/removed.
* New Wazuh version installed on **your chosen drive** (**C:** or **D:**).
* Certs generated, ports verified, and dashboard accessible.
