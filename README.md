# APIPark Local Setup Guide

Complete guide for setting up APIPark on your local machine using Docker.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Initial Setup](#initial-setup)
3. [Configuration](#configuration)
4. [Starting APIPark](#starting-apipark)
5. [Access URLs & Credentials](#access-urls--credentials)
6. [Post-Installation Configuration](#post-installation-configuration)
7. [Important Notes](#important-notes)
8. [Known Limitations](#known-limitations)

---

## Prerequisites

### Required Software

- **Docker Desktop** (or Docker Engine + Docker Compose)
  - Download: https://www.docker.com/products/docker-desktop
  - Verify installation:
    ```bash
    docker --version
    docker compose version
    ```

### System Requirements

- **OS**: macOS
- **RAM**: Minimum 4GB available
- **Disk**: Minimum 10GB free space
- **Network**: Internet connection for pulling Docker images

---

## Initial Setup

### Step 1: Get Your Local IP Address

**CRITICAL:** APIPark requires your **actual host IP address**, not `localhost` or `127.0.0.1`.

**On macOS:**

```bash
ifconfig | grep "inet " | grep -v 127.0.0.1
```

**Example Output:**
```
inet 192.168.1.109 netmask 0xffffff00 broadcast 192.168.1.255
```

**Your IP:** `192.168.1.109` (use this in the next step)

**⚠️ IMPORTANT:** 
- Do NOT use `localhost` or `127.0.0.1`
- Use the actual IP address from your local network (e.g., `192.168.x.x` or `10.0.x.x`)
- This IP will be used in `config.yml`

---

## Configuration

### Step 2: Update config.yml

Navigate to the project directory and edit the configuration file:

```bash
cd /path/to/APIPark
nano scripts/config.yml
```

**Replace `<IP>` placeholders with your actual IP address:**

```yaml
version: 2
client:
  advertise_urls:
    - http://192.168.1.109:19400  # ← Replace with YOUR IP
  listen_urls:
    - http://0.0.0.0:9400

gateway:
  advertise_urls:
    - http://192.168.1.109:18099  # ← Replace with YOUR IP
    - https://192.168.1.109:18099 # ← Replace with YOUR IP
  listen_urls:
    - https://0.0.0.0:8099
    - http://0.0.0.0:8099

peer:
  listen_urls:
    - http://0.0.0.0:9401
  advertise_urls:
    - http://192.168.1.109:19401  # ← Replace with YOUR IP
```

**What to Replace:**
- Line 6: `http://192.168.1.105:19400` → `http://YOUR_IP:19400`
- Line 14: `http://192.168.1.105:18099` → `http://YOUR_IP:18099`
- Line 15: `https://192.168.1.105:18099` → `https://YOUR_IP:18099`
- Line 23: `http://192.168.1.105:19401` → `http://YOUR_IP:19401`

**Save the file** (Ctrl+O, Enter, Ctrl+X in nano)

---

## Starting APIPark

### Step 3: Start All Services

From the project root directory:

```bash
docker compose -f scripts/docker-compose.yml up -d
```

**Expected Output:**
```
[+] Running 9/9
 ✔ Network apipark                Created
 ✔ Container apipark-mysql        Started
 ✔ Container apipark-redis        Started
 ✔ Container apipark-influxdb     Started
 ✔ Container apipark-loki         Started
 ✔ Container apipark-nsq          Started
 ✔ Container apipark-grafana      Started
 ✔ Container apipark              Started
 ✔ Container apipark-apinto       Started
```

### Step 4: Verify Services Are Running

```bash
docker compose -f scripts/docker-compose.yml ps
```

**All services should show status: `Up`**
---

## Access URLs & Credentials

### Access URLs

Once all services are running, access the following URLs:

| Service | URL | Purpose |
|---------|-----|---------|
| **APIPark UI** | http://localhost:18288 | Main admin interface for managing APIs, services, teams |
| **Apinto Gateway** | http://localhost:18099 | API Gateway - where your published APIs are accessible |
| **Cluster Address** | http://localhost:19400 | Admin/Cluster API for automation and system management |
| **Grafana** | http://localhost:3000 | Log visualization and monitoring dashboards |
| **InfluxDB** | http://localhost:8086 | Time-series database for metrics storage |

### Default Credentials

| Service | Username | Password | Additional Info |
|---------|----------|----------|-----------------|
| **APIPark Admin** | admin | `12345678` | |
| **MySQL** | root | `123456` | |
| **Redis** | N/A | `123456` | |
| **InfluxDB** | admin | `Key123qaz` | Token: `apipark-influxdb-token-2024` |
| **Grafana** | N/A | N/A | |

---

## Post-Installation Configuration

### Step 5: Update Gateway IP in Admin Panel

**CRITICAL STEP:** After starting APIPark, you must configure the gateway IP in the admin panel.

1. **Login to APIPark UI:**
   - Go to http://localhost:18288
   - Login with `admin` / `12345678`

2. **Navigate to Cluster Settings:**
   ```
   System Settings → API Gateway → Edit Settings → Cluster Address
   ```

3. **Update Gateway Address:**
   - **Cluster Address:** `http://YOUR_IP:19400` (e.g., `http://192.168.1.109:19400`)
   - Click **Save**

### Step 6: Configure AI Model Settings (If Using AI APIs)

**IMPORTANT:** When configuring AI models, you must **remove the `json_schema` key** from the model configuration.

**Correct Configuration Example:**

```json
{
  "frequency_penalty": null,
  "max_tokens": 512,
  "presence_penalty": null,
  "response_format": null,
  "temperature": 0.5,
  "top_p": null
}
```

**❌ INCORRECT (will cause errors):**
```json
{
  "json_schema": {...},  ← Remove this
  "max_tokens": 512,
  "temperature": 0.5
}
```
---

## Important Notes

### IP Address Changes

**⚠️ CRITICAL:** If your local IP address changes (e.g., after reconnecting to WiFi, DHCP renewal), you MUST:

1. **Stop all containers:**
   ```bash
   docker compose -f scripts/docker-compose.yml down -v
   ```

2. **Get new IP address:**
   ```bash
   ifconfig | grep "inet " | grep -v 127.0.0.1
   ```

3. **Update `scripts/config.yml`:**
   - Replace all occurrences of old IP with new IP
   - Update lines 6, 14, 15, and 23

4. **Restart containers:**
   ```bash
   docker compose -f scripts/docker-compose.yml up -d
   ```

5. **Update Gateway IP in Admin Panel:**
   - Login to APIPark UI
   - Go to System Settings → API Gateway → Edit Settings → Cluster Address
   - Update Cluster Address with new IP
   - Save changes
---

## Known Limitations

### 1. PII Handling
**Issue:** Personally Identifiable Information (PII) is still sent to the LLM

**Details:**
- Data Masking feature works only at the **response level**, not request level
- PII in requests is sent to AI providers (OpenAI, Claude, etc.) **unmasked**
- Data Masking currently works only for **REST APIs**, not AI APIs

**Impact:**
- Cannot protect PII before it reaches LLMs
- Compliance risk for sensitive data
- Response masking has limited practical value

**Workaround:**
- Implement PII redaction before APIPark
- Use external pre-processing layer
- Avoid sending PII to AI APIs

### 2. Intermittent Errors
Occasional sudden timeouts may occur (e.g., `lookup api.openai.com: i/o timeout`), particularly when updating prompts or CMS items.

### 3. Non-Streaming Responses
Errors may occur when handling non-streaming responses from AI providers. Some AI APIs may fail when streaming is disabled, resulting in response parsing errors or incomplete responses.

### 4. Feature Completeness
The open-source repository is still under active development. Some features are incomplete, bugs exist, and documentation may be outdated. Pin to a specific version (currently using `v1.9.0-beta`) and test thoroughly before production use.

### 5. Log Processing Delays
The built-in logging system is not real-time due to its producer-queue-consumer architecture. Logs may appear minutes after API calls, making real-time debugging difficult.

### 6. Backend Language
The backend is implemented in GoLang. Modifying source code requires Go knowledge and compilation. This is not a limitation but just want to make a note of it.

---

**Last Updated:** 2025-10-10  
**APIPark Version:** v1.9.0-beta
