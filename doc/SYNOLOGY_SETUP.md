# Synology NAS Setup Guide for aMule Docker

This guide covers the complete setup to get aMule running with **High ID** on a Synology NAS (DSM 7.x) using Container Manager with bridge networking, including DSM Firewall rules and Orange Livebox router port forwarding.

> **Goal**: Get a High ID on ED2K servers and an open (green) Kad connection, meaning your aMule container is fully reachable from the internet.

## Table of Contents

- [Overview: Ports Required](#overview-ports-required)
- [Step 1: Container Manager Configuration](#step-1-container-manager-configuration)
- [Step 2: DSM Firewall Configuration](#step-2-dsm-firewall-configuration)
- [Step 3: Orange Livebox Port Forwarding](#step-3-orange-livebox-port-forwarding)
- [Verification Checklist](#verification-checklist)
- [Troubleshooting](#troubleshooting)

---

## Overview: Ports Required

aMule needs the following ports open end-to-end (from the internet to the container):

| Port | Protocol | Purpose | Required for |
|------|----------|---------|-------------|
| 4662 | TCP | ED2K client-to-client transfers | High ID |
| 4662 | UDP | ED2K UDP (Emulerr compatibility) | Emulerr |
| 4665 | UDP | ED2K global server search (auto: Port + 3) | High ID |
| 4672 | UDP | Kad network / extended ED2K protocol | Kad Open status |
| 4711 | TCP | Web UI | LAN access only |
| 4712 | TCP | Remote GUI (EC protocol) | LAN access only |

> **Important**: Ports 4711 and 4712 are for local management only. Do **not** forward them on your router.

The traffic path is: **Internet → Livebox (NAT) → Synology (Firewall) → Docker (Bridge) → aMule Container**

Each layer must allow the traffic through. If any layer blocks or misconfigures a port, you will get Low ID / Firewalled status.

---

## Step 1: Container Manager Configuration

### Option A: Using docker-compose (recommended)

1. Open **Container Manager** > **Project** > **Create**
2. Set a project name (e.g., `amule`)
3. Set the path to your compose file or paste the following:

```yaml
---
services:
  amule:
    image: ngosang/amule
    container_name: amule
    environment:
      - PUID=1026          # Your Synology user UID (check with: id <username>)
      - PGID=100           # Your Synology user GID (typically 100 = users)
      - TZ=Europe/Paris    # Set to your timezone
      - GUI_PWD=your_gui_password
      - WEBUI_PWD=your_webui_password
      - MOD_AUTO_RESTART_ENABLED=true
      - MOD_AUTO_RESTART_CRON=0 6 * * *
      - MOD_AUTO_SHARE_ENABLED=false
      - MOD_FIX_KAD_GRAPH_ENABLED=true
      - MOD_FIX_KAD_BOOTSTRAP_ENABLED=true
    ports:
      - "4711:4711"        # Web UI (LAN only, do NOT forward on router)
      - "4712:4712"        # Remote GUI (LAN only, do NOT forward on router)
      - "4662:4662"        # ED2K TCP — must be forwarded on router
      - "4662:4662/udp"    # ED2K UDP — needed by Emulerr
      - "4665:4665/udp"    # ED2K global search — must be forwarded on router
      - "4672:4672/udp"    # Kad / ED2K extended — must be forwarded on router
    volumes:
      - /volume1/docker/amule/config:/home/amule/.aMule
      - /volume1/docker/amule/incoming:/incoming
      - /volume1/docker/amule/temp:/temp
    restart: unless-stopped
```

4. Click **Next**, then **Done** to deploy

### Option B: Using Container Manager UI

If you prefer using the Container Manager graphical interface:

1. **Container Manager** > **Image** > Download `ngosang/amule`
2. **Container** > **Create** from the `ngosang/amule` image
3. **General Settings**:
   - Container name: `amule`
   - Enable auto-restart: ✅
4. **Advanced Settings** > **Environment**:
   - Add each variable listed in the compose example above
5. **Port Settings** — this is critical:

   | Local Port | Container Port | Protocol |
   |-----------|---------------|----------|
   | 4711 | 4711 | TCP |
   | 4712 | 4712 | TCP |
   | 4662 | 4662 | TCP |
   | 4662 | 4662 | UDP |
   | 4665 | 4665 | UDP |
   | 4672 | 4672 | UDP |

   > ⚠️ **Critical**: Local port and container port must be the **same number**. Synology Container Manager sometimes auto-assigns random local ports — you must change them manually to match. Mismatched ports (e.g., local 32662 → container 4662) will result in Low ID.

6. **Volume Settings**:

   | Host path | Mount path |
   |-----------|-----------|
   | `/volume1/docker/amule/config` | `/home/amule/.aMule` |
   | `/volume1/docker/amule/incoming` | `/incoming` |
   | `/volume1/docker/amule/temp` | `/temp` |

7. Click **Done** to create the container

### Finding Your PUID and PGID

Connect to your Synology via SSH and run:

```bash
id <your_username>
```

Example output:
```
uid=1026(myuser) gid=100(users) groups=100(users),101(administrators)
```

Use `PUID=1026` and `PGID=100` in your configuration.

### Creating the Volume Directories

```bash
mkdir -p /volume1/docker/amule/config
mkdir -p /volume1/docker/amule/incoming
mkdir -p /volume1/docker/amule/temp
chown -R 1026:100 /volume1/docker/amule
```

---

## Step 2: DSM Firewall Configuration

If the DSM Firewall is enabled (**Control Panel** > **Security** > **Firewall**), you must allow traffic on aMule's ports. By default, DSM Firewall may block incoming connections to Docker container ports.

### Check if Firewall is Enabled

1. Open **Control Panel** > **Security** > **Firewall**
2. If "Enable firewall" is checked, you need to add rules

### Add Firewall Rules

1. Click **Edit Rules** on the active firewall profile
2. Create rules to **allow** the following ports:

#### Rule 1: aMule ED2K TCP

| Setting | Value |
|---------|-------|
| Ports | Custom: `4662` |
| Protocol | TCP |
| Source IP | All |
| Action | Allow |

#### Rule 2: aMule UDP Ports

| Setting | Value |
|---------|-------|
| Ports | Custom: `4662, 4665, 4672` |
| Protocol | UDP |
| Source IP | All |
| Action | Allow |

#### Rule 3: aMule Web UI (optional, LAN only)

| Setting | Value |
|---------|-------|
| Ports | Custom: `4711, 4712` |
| Protocol | TCP |
| Source IP | Specific IP: `192.168.1.0/255.255.255.0` (your LAN subnet) |
| Action | Allow |

> **Security tip**: Restrict the Web UI rule (Rule 3) to your local network only. Never forward ports 4711/4712 to the internet.

3. Ensure these **Allow** rules are **above** any catch-all **Deny** rule
4. Click **OK** then **Apply**

### Important: Rule Order

DSM Firewall rules are evaluated top-to-bottom. If you have a default "deny all" rule at the bottom:

```
1. Allow  TCP  4662        All          ← aMule ED2K
2. Allow  UDP  4662,4665,4672  All      ← aMule UDP
3. Allow  TCP  4711,4712   192.168.1.0  ← Web UI (LAN)
4. Allow  ...  ...         ...          ← your other rules
5. Deny   All  All         All          ← default deny
```

The aMule allow rules **must** be above the deny-all rule.

---

## Step 3: Orange Livebox Port Forwarding

The Orange Livebox needs NAT/PAT rules to forward incoming internet traffic to your Synology NAS.

### Find Your Synology's Local IP

On your Synology: **Control Panel** > **Network** > **Network Interface** — note the IP address (e.g., `192.168.1.50`).

> **Tip**: Assign a static IP to your Synology (or use a DHCP reservation in the Livebox) so the port forwarding rules remain valid.

### Configure NAT/PAT Rules on the Livebox

1. Open the Livebox admin panel: navigate to `http://192.168.1.1` in your browser
2. Log in (default credentials are on the sticker under the Livebox)
3. Navigate to: **Réseau** (Network) > **NAT/PAT**
4. Add the following rules:

#### Rule 1: aMule ED2K TCP

| Field | Value |
|-------|-------|
| Application/Service | Custom / `aMule_TCP` |
| Internal port | 4662 |
| External port | 4662 |
| Protocol | TCP |
| Device / IP | Your Synology (e.g., `192.168.1.50`) |
| Enabled | ✅ |

#### Rule 2: aMule ED2K UDP

| Field | Value |
|-------|-------|
| Application/Service | Custom / `aMule_UDP_4662` |
| Internal port | 4662 |
| External port | 4662 |
| Protocol | UDP |
| Device / IP | Your Synology (e.g., `192.168.1.50`) |
| Enabled | ✅ |

#### Rule 3: aMule Global Search UDP

| Field | Value |
|-------|-------|
| Application/Service | Custom / `aMule_UDP_4665` |
| Internal port | 4665 |
| External port | 4665 |
| Protocol | UDP |
| Device / IP | Your Synology (e.g., `192.168.1.50`) |
| Enabled | ✅ |

#### Rule 4: aMule Kad UDP

| Field | Value |
|-------|-------|
| Application/Service | Custom / `aMule_UDP_4672` |
| Internal port | 4672 |
| External port | 4672 |
| Protocol | UDP |
| Device / IP | Your Synology (e.g., `192.168.1.50`) |
| Enabled | ✅ |

5. Click **Save** / **Enregistrer** for each rule

> **Note**: On some Livebox models (Livebox 4, 5, 6), the interface may differ slightly. Look for **NAT/PAT** or **Redirection de ports** under the network settings.

### Orange Livebox Firewall

The Livebox has its own built-in firewall separate from NAT/PAT:

1. Go to **Réseau** (Network) > **Pare-feu** (Firewall)
2. Set the firewall level to **Medium** (Moyen) or **Custom** (Personnalisé)
3. If set to **Custom**, ensure the forwarded ports (4662 TCP/UDP, 4665 UDP, 4672 UDP) are not blocked
4. If set to **High** (Élevé), NAT/PAT forwarding rules may be overridden by the firewall — switch to **Medium**

> **Important**: The Livebox "High" firewall level can silently block forwarded ports even if NAT/PAT rules are correctly set. Use "Medium" or "Custom" level.

---

## Verification Checklist

After completing all three steps, verify your setup:

### 1. Check Container Logs

In Container Manager, view the `amule` container logs. You should see:

```
Effective port configuration:
  ED2K TCP:               4662 (Port)
  ED2K UDP:               4662 (Port, needed by Emulerr)
  ED2K Global Search UDP: 4665 (Port + 3, auto-derived)
  Kad/ED2K UDP:           4672 (UDPPort)
  WebUI TCP:              4711
  EC (Remote GUI) TCP:    4712
```

Verify these match your Docker port mappings and router forwarding rules.

### 2. Check aMule Web UI

1. Open `http://<synology-ip>:4711` in your browser
2. Log in with your `WEBUI_PWD` password
3. Check the server connection status:
   - **High ID** = ports are correctly forwarded (good ✅)
   - **Low ID** = something is blocking the ports (bad ❌)
4. Check Kad network:
   - **Connected** (green) = Kad ports open (good ✅)
   - **Firewalled** (orange) = UDP ports blocked (bad ❌)

### 3. Test Port Connectivity

From an external network (or using an online port checker):

- Test TCP 4662: should be **open**
- Test UDP 4665: should be **open**
- Test UDP 4672: should be **open**

Online port check tools:
- https://www.yougetsignal.com/tools/open-ports/
- https://portchecker.co/

> **Note**: aMule must be running for port checks to succeed — the ports are not open unless the application is actively listening.

### 4. Quick Diagnostic Table

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Low ID on ED2K servers | TCP 4662 not forwarded | Check Livebox NAT/PAT + DSM Firewall |
| Kad "Firewalled" (orange) | UDP 4672 not forwarded | Check Livebox NAT/PAT + DSM Firewall |
| Kad not connecting at all | nodes.dat missing | Set `MOD_FIX_KAD_BOOTSTRAP_ENABLED=true` |
| Web UI not accessible | Port 4711 not mapped in Docker | Check Container Manager port settings |
| Ports shown as random numbers in Container Manager | Auto-assigned local ports | Manually set local ports to match container ports |
| Everything configured but still Low ID | Livebox firewall on "High" | Switch Livebox firewall to "Medium" |

---

## Troubleshooting

### "I have Low ID but all ports seem configured"

1. **Restart the container** after any port configuration change
2. **Wait 2-3 minutes** for aMule to establish connections and get its ID
3. **Check the Livebox firewall level** — "High" blocks forwarded ports
4. **Verify DSM Firewall rule order** — Allow rules must be above Deny rules
5. **Delete the existing `amule.conf`** and restart to regenerate with correct ports:
   ```bash
   rm /volume1/docker/amule/config/amule.conf
   # Then restart the container
   ```

### "Kad stays Firewalled even with High ID on ED2K"

Kad uses UDP port 4672 (or your `AMULE_UDP_PORT`). This is a different port from ED2K TCP. Ensure:
- UDP 4672 is forwarded on the Livebox
- UDP 4672 is allowed in DSM Firewall
- Docker maps `4672:4672/udp`

### "Port check shows ports as closed"

- Ensure aMule is **running** when you test
- Check your ISP is not blocking these ports (some ISPs block common P2P ports)
- If Orange blocks certain ports, try custom ports using `AMULE_PORT` and `AMULE_UDP_PORT` environment variables

### "Container Manager shows wrong ports after editing"

If Container Manager reverts your port settings:
1. Stop the container
2. Delete it
3. Recreate it with the correct port mappings
4. Or switch to the docker-compose method (recommended), which is more reliable for port configuration

### Using Custom Ports (if default ports are blocked)

If your ISP blocks the default ports, you can use custom ports. Example with port 53123:

```yaml
environment:
  - AMULE_PORT=53123
  - AMULE_UDP_PORT=53127
ports:
  - "4711:4711"
  - "4712:4712"
  - "53123:53123"        # ED2K TCP
  - "53123:53123/udp"    # ED2K UDP (Emulerr)
  - "53126:53126/udp"    # ED2K global search (53123 + 3)
  - "53127:53127/udp"    # Kad UDP
```

Then update your Livebox NAT/PAT rules and DSM Firewall rules to use these new port numbers instead.
