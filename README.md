# npm-opencloud-bf

CrowdSec collection to detect bruteforce attacks on **Opencloud/LibreGraph** behind **Nginx Proxy Manager**.

## 📋 Description

Opencloud uses **LibreGraph Identity Management** as its Identity Provider (IDP). Unlike traditional authentication systems that return HTTP 401/403 on failed login attempts, LibreGraph returns:
- **HTTP 204** (No Content) when login **fails**
- **HTTP 200** (OK) when login **succeeds**

This makes it impossible to detect bruteforce attacks using standard HTTP status code monitoring. This collection solves that problem by monitoring the specific authentication endpoint and detecting repeated HTTP 204 responses.

---

## ⚙️ Requirements

### 1. Nginx Proxy Manager
Your Opencloud instance must be behind **Nginx Proxy Manager** with access logs enabled.

### 2. CrowdSec nginx-proxy-manager collection
This collection **depends** on the official CrowdSec parser for Nginx Proxy Manager:

👉 **[crowdsecurity/nginx-proxy-manager](https://app.crowdsec.net/hub/author/crowdsecurity/collections/nginx-proxy-manager)**

Install it first:
```bash
sudo cscli collections install crowdsecurity/nginx-proxy-manager
```

---

## 🚀 Installation

### Step 1: Install dependencies
```bash
sudo cscli collections install crowdsecurity/nginx-proxy-manager
```

### Step 2: Install this scenario
```bash
# Download the scenario file
sudo wget -O /etc/crowdsec/scenarios/npm-opencloud-bf.yaml \
  https://raw.githubusercontent.com/JGeek00/crowdsec-npm-opencloud-bf/main/scenarios/npm-opencloud-bf.yaml

# Reload CrowdSec
sudo systemctl reload crowdsec
```

### Step 3: Configure log acquisition
Ensure your `acquis.yaml` (or a file in `acquis.d/`) includes Nginx Proxy Manager logs:

```yaml
# /etc/crowdsec/acquis.yaml or /etc/crowdsec/acquis.d/nginx-proxy-manager.yaml
filenames:
  - /var/log/shared/*.log
  - /var/log/shared/**/*.log
labels:
  type: nginx-proxy-manager
```

Adjust the path to match your NPM log location.

### Step 4: Verify installation
```bash
sudo cscli scenarios list | grep npm-opencloud
```

You should see:
```
jgeek00/npm-opencloud-bf    ✓    installed
```

---

## 🎯 Detection Logic

| Parameter | Value | Description |
|-----------|-------|-------------|
| **Endpoint** | `/signin/v1/identifier/_/logon` | LibreGraph authentication endpoint |
| **Method** | `POST` | Only POST requests are monitored |
| **Failure indicator** | HTTP `204` | LibreGraph returns 204 on failed login |
| **Threshold** | `5 attempts` | Number of failed attempts before ban |
| **Time window** | `10 seconds` | All attempts must occur within this window |
| **Ban duration** | `1 minute` | Default ban time (configurable) |

---

## 🧪 Testing

### 1. Check metrics
```bash
sudo cscli metrics
```

Look for `jgeek00/npm-opencloud-bf` in the scenario metrics table.

### 2. Simulate an attack
1. Open your Opencloud login page
2. Attempt to login with **incorrect credentials** 5 times **within 10 seconds**
3. Check if your IP gets banned:

```bash
sudo cscli decisions list
```

You should see:
```
┌────┬──────────┬────────────────┬─────────────────────────┬────────┐
│ ID │ Source   │ IP             │ Reason                  │ Action │
├────┼──────────┼────────────────┼─────────────────────────┼────────┤
│ 1  │ cscli    │ 203.0.113.55   │ jgeek00/npm-opencloud-bf│ ban    │
└────┴──────────┴────────────────┴─────────────────────────┴────────┘
```

### 3. Remove test ban
```bash
sudo cscli decisions delete --ip YOUR_IP
```

---

## 🔧 Configuration

### Adjust detection sensitivity

Edit `/etc/crowdsec/scenarios/npm-opencloud-bf.yaml`:

```yaml
# Allow 10 attempts instead of 5
capacity: 10

# Increase time window to 30 seconds
leakspeed: "30s"

# Ban for 1 hour instead of 1 minute
blackhole: 1h
```

After changes:
```bash
sudo systemctl reload crowdsec
```

### Whitelist your network

To avoid banning yourself during testing:

```bash
sudo nano /etc/crowdsec/parsers/s02-enrich/whitelist-local.yaml
```

```yaml
name: crowdsecurity/whitelists
description: "Whitelist local network"
whitelist:
  reason: "local network"
  ip:
    - "192.168.0.0/16"
    - "10.0.0.0/8"
    - "YOUR_PUBLIC_IP/32"
```

---

## 📊 How it works

```
┌─────────────┐      ┌─────────────────┐      ┌──────────────┐
│   Attacker  │─────▶│ Nginx Proxy     │─────▶│  Opencloud   │
│             │      │ Manager (NPM)   │      │ LibreGraph   │
└─────────────┘      └─────────────────┘      └──────────────┘
                              │
                              │ Logs: POST /signin/v1/identifier/_/logon → 204
                              ▼
                     ┌─────────────────┐
                     │   CrowdSec      │
                     │  npm-opencloud  │
                     │   scenario      │
                     └─────────────────┘
                              │
                              │ 5 attempts in 10s
                              ▼
                     ┌─────────────────┐
                     │   IP BANNED     │
                     └─────────────────┘
```

1. **User attempts login** → Nginx Proxy Manager forwards to Opencloud
2. **LibreGraph validates credentials** → Returns HTTP 204 on failure
3. **NPM logs the request** → Writes to `proxy-host-X_access.log`
4. **CrowdSec reads the log** → Parses with `crowdsecurity/nginx-proxy-manager`
5. **Scenario matches filter** → Counts attempts per IP
6. **Threshold exceeded** → IP gets banned for 1 minute

---

## 🆚 Comparison with other approaches

### This collection (npm-opencloud-bf)
✅ Works with dockerized Opencloud  
✅ No need to configure LibreGraph trusted proxies  
✅ Uses reliable NPM logs with real client IPs  
✅ Easy to deploy and maintain  

### Alternative: Parse LibreGraph internal logs
❌ Requires configuring `OCIS_TRUSTED_PROXIES`  
❌ More complex setup in Docker environments  
❌ LibreGraph logs show `127.0.0.1` by default  

---

## 🐛 Troubleshooting

### Scenario not detecting events

Check if the parser is working:
```bash
tail -n 1 /var/log/shared/proxy-host-X_access.log | sudo cscli explain --type nginx-proxy-manager -f -
```

Look for:
- `✓ crowdsecurity/nginx-proxy-manager` in s01-parse stage
- `evt.Meta.http_status: 204`
- `evt.Meta.http_path: /signin/v1/identifier/_/logon`
- `🔥 MATCH` in Scenarios section

### No decisions created after 5 attempts

Check metrics:
```bash
sudo cscli metrics
```

Look at the `npm-opencloud-bf` row:
- **Poured** should be ≥ 5
- **Overflows** should be ≥ 1

If `Overflows` is still `-`, the attempts might be too slow (>10 seconds apart).

### CrowdSec fails to start

Check syntax:
```bash
sudo systemctl status crowdsec
```

Validate scenario file:
```bash
sudo cscli scenarios inspect jgeek00/npm-opencloud-bf
```

---

## 📚 Resources

- **Opencloud**: https://github.com/owncloud/ocis
- **LibreGraph**: https://github.com/libregraph
- **CrowdSec Documentation**: https://docs.crowdsec.net/
- **Nginx Proxy Manager Collection**: https://app.crowdsec.net/hub/author/crowdsecurity/collections/nginx-proxy-manager

---

## 🤝 Contributing

Issues and pull requests are welcome!

---

## 📝 License

MIT License - See [LICENSE](LICENSE) file for details

---

## 👤 Author

**JGeek00**

- GitHub: [@JGeek00](https://github.com/JGeek00)
- Repository: [crowdsec-npm-opencloud-bf](https://github.com/JGeek00/crowdsec-npm-opencloud-bf)