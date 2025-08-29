# Wazuh Docker (Windows) — Replace **v4.7.4** with a New Version (D:)

This markdown starts **from stopping/removing the current Wazuh stack** and goes **down to installing your new target version**.

> Run all commands in **PowerShell (Administrator)**.

---

## 1) Stop & remove the current Wazuh (v4.7.4) stack

If you’re in the original compose folder you used to start 4.7.4:

```powershell
# from that folder
docker compose down
```

If you don’t have the old folder handy, remove by image/ancestor (specific to **4.7.4**):

```powershell
# Stop/remove any containers based on 4.7.4 images
docker ps -a -q --filter ancestor=wazuh/wazuh-manager:4.7.4   | % { docker rm -f $_ }
docker ps -a -q --filter ancestor=wazuh/wazuh-indexer:4.7.4   | % { docker rm -f $_ }
docker ps -a -q --filter ancestor=wazuh/wazuh-dashboard:4.7.4 | % { docker rm -f $_ }
```

**Free common ports (1514/1515) if still busy:**

```powershell
docker ps --format "table {{.Names}}\t{{.Ports}}" | findstr 1514
# If a container shows up using 1514/1515, stop & remove it:
# docker stop <name>; docker rm <name>
```

**If a Windows process owns the port (not a container):**

```powershell
Get-NetTCPConnection -LocalPort 1514 -ErrorAction SilentlyContinue | Select LocalAddress,LocalPort,State,OwningProcess
# Then identify/stop the process if safe:
Get-Process -Id <PID>
# Stop-Process -Id <PID> -Force
```

---

## 2) (Optional but recommended for a clean slate) Remove old Wazuh volumes

> ⚠️ Destructive — erases saved settings/data. Skip if you intend to keep data.

```powershell
docker volume ls -q --filter "name=wazuh" | % { docker volume rm $_ }
```

---

## 3) (Optional) Remove the old v4.7.4 images to free disk

```powershell
docker rmi wazuh/wazuh-manager:4.7.4 `
          wazuh/wazuh-indexer:4.7.4 `
          wazuh/wazuh-dashboard:4.7.4

# (Optional) If present and you want to repull later
# docker rmi wazuh/wazuh-certs-generator:0.0.2
```

---

## 4) Install your **new** Wazuh version on \**D:\**

Set the versions/paths you want up front:

```powershell
$NEW_VER = "vX.Y.Z"           # e.g., "v4.9.3" or "v4.12.0"
$BASE    = "D:\wazuh-$NEW_VER"
```

Create the working folder and clone the exact tag:

```powershell
mkdir $BASE
cd    $BASE

git clone https://github.com/wazuh/wazuh-docker.git -b $NEW_VER
cd .\wazuh-docker\single-node
```

### 4.1) Generate TLS certificates

First make sure `config\certs.yml` is a **file** (not a folder):

```powershell
Get-Item .\config\certs.yml | Format-List Mode,Name,Length,FullName
# If Mode shows 'd----', fix it:
# Remove-Item -Recurse -Force .\config\certs.yml
# git checkout -- .\config\certs.yml
```

Generate the certs:

```powershell
docker compose -f .\generate-indexer-certs.yml run --rm generator
```

Quick check:

```powershell
Get-ChildItem .\config\wazuh_indexer_ssl_certs -Recurse
```

### 4.2) (Optional) Use a unique project name

Prevents container-name clashes with other stacks.

```powershell
$env:COMPOSE_PROJECT_NAME = "wazuh-$NEW_VER"
```

### 4.3) (Optional) Preempt port conflicts by remapping

If 1514/1515 might be busy on your PC, edit `docker-compose.yml` (service `wazuh.manager`) and change:

```yaml
ports:
  - "15140:1514"
  - "15140:1514/udp"
  - "15150:1515"
  - "5140:514/udp"
  - "55000:55000"
# dashboard example if needed:
#  - "8443:443"
# or if your compose uses 5601:
#  - "56010:5601"
```

### 4.4) Start the new stack

```powershell
docker compose up -d
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

> First boot can take a bit while the indexer initializes; transient “not ready yet” messages are normal.

### 4.5) Open the dashboard

Find the mapped dashboard port:

```powershell
docker ps --format "table {{.Names}}\t{{.Ports}}" | findstr dashboard
```

* If you see `0.0.0.0:5601->5601/tcp` → **[http://localhost:5601](http://localhost:5601)**
* If you see `0.0.0.0:443->443/tcp` → **[https://localhost](https://localhost)** (self-signed cert; accept warning)

Log in with the default credentials for that version and change the password afterward.

---

## 5) Troubleshooting quickies

**Port already allocated (1514/1515):**

```powershell
docker ps --format "table {{.Names}}\t{{.Ports}}" | findstr 1514
# stop/remove the offender, or remap ports in docker-compose.yml (see 4.3) and:
docker compose up -d
```

**Cert generator error mentioning `/config/certs.yml`:**

* Ensure `.\config\certs.yml` is a **file** and exists, then re-run:

```powershell
docker compose -f .\generate-indexer-certs.yml run --rm generator
```

**“Dashboard not ready yet” at first start:**

* Wait a bit; check logs:

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

### ✅ You’re done

* Old **v4.7.4** stack stopped/removed.
* New Wazuh version installed on \**D:\** with certs and ports sorted.
* Use the dashboard port shown by `docker ps` to log in.
