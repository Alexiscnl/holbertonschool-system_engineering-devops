# 2. Secured and Monitored Web Infrastructure

A three-server web infrastructure for **www.foobar.com** that is **secured**, serves **encrypted traffic (HTTPS)**, and is **monitored**.

---

## 🌐 Diagram – Flowchart

```mermaid
flowchart TB
    U["User (Browser)"] -->|DNS resolves www.foobar.com| LB["HAProxy Load Balancer"]

    %% Firewalls at three layers (network segmentation)
    FW_EDGE["Firewall #1 (edge/public)"]
    FW_APP["Firewall #2 (app tier)"]
    FW_DB["Firewall #3 (db tier)"]

    U --> FW_EDGE --> LB

    subgraph S1["Server #1"]
      N1["Nginx web"] --> A1["App server"]
      A1 --> D1[("MySQL Primary")]
      MC1["Monitoring client"]
    end

    subgraph S2["Server #2"]
      N2["Nginx web"] --> A2["App server"]
      A2 --> D2[("MySQL Replica")]
      MC2["Monitoring client"]
    end

    subgraph S3["Server #3"]
      N3["Nginx web"] --> A3["App server"]
      A3 --> D3[("MySQL Replica")]
      MC3["Monitoring client"]
    end

    LB -->|HTTPS 443| N1
    LB -->|HTTPS 443| N2
    LB -->|HTTPS 443| N3

    %% App/DB tier firewalls
    N1 --> FW_APP
    N2 --> FW_APP
    N3 --> FW_APP

    FW_APP --> D1
    FW_APP --> D2
    FW_APP --> D3

    %% DB firewall guarding intra-DB communication
    D1 <-->|Async replication| D2
    D1 <-->|Async replication| D3
    D1 --- FW_DB
    D2 --- FW_DB
    D3 --- FW_DB

    %% Monitoring backend (SaaS like Sumo Logic/Datadog/Prometheus remote)
    subgraph MON["Monitoring Service"]
      MS["Collector / SaaS backend"]
    end
    MC1 ==> MS
    MC2 ==> MS
    MC3 ==> MS
```
```mermaid
sequenceDiagram
    participant User as User (Browser)
    participant DNS as DNS
    participant FW1 as Firewall #1 (edge)
    participant LB as HAProxy (TLS)
    participant FW2 as Firewall #2 (app)
    participant N as Nginx (web)
    participant A as App (backend)
    participant FW3 as Firewall #3 (db)
    participant DBP as MySQL Primary
    participant DBR as MySQL Replica
    participant MS as Monitoring Service

    User->>DNS: Resolve www.foobar.com
    DNS-->>User: IP of LB
    User->>FW1: HTTPS 443
    FW1-->>LB: Allow (443)
    User->>LB: TLS handshake + GET /
    LB->>N: Forward HTTPS (443) (mutual or server-only TLS)
    N->>A: Proxy dynamic request
    A->>FW3: Request DB access (3306)
    FW3-->>DBP: Allow from app tier (3306)
    A->>DBP: Write / critical read
    DBP-->>A: Results
    A-->>N: Response (HTML/JSON)
    N-->>LB: Upstream response
    LB-->>User: 200 OK over HTTPS

    Note over DBP,DBR: Primary streams binlog to Replicas (async)
    Note over MS: Agents push logs/metrics/traces to monitoring backend
```
---

## 📄 Explanation

### 1) User Request, HTTPS, and Firewalls
1. The user opens a browser and types `www.foobar.com`.
2. The browser asks the DNS to resolve the domain name into an IP address.
3. The DNS replies with the IP address of the **load balancer**.
4. The browser establishes an **HTTPS connection** with the load balancer using an **SSL certificate**.
5. **Firewall #1 (edge firewall)** allows only port 443 (HTTPS) and blocks everything else.
6. The load balancer forwards the request to one of the web servers over HTTPS.
7. **Firewall #2 (application tier firewall)** only allows traffic from the web servers to the application servers.
8. **Firewall #3 (database firewall)** only allows database connections from the application servers.

---

### 2) Backend Servers
1. **Nginx (web server):**
   - Terminates HTTPS (if TLS is end-to-end, Nginx also uses HTTPS internally).
   - Serves static files and forwards dynamic requests to the application server.
2. **Application server:**
   - Runs the business logic (code).
   - If data is required, it queries the database.
3. **MySQL Primary–Replica cluster:**
   - The **Primary** database handles all writes.
   - The **Replicas** receive asynchronous updates from the Primary and can serve read-only queries.

---

### 3) HTTPS and SSL Certificate
- **Why use HTTPS:**
  - Encrypts traffic to protect sensitive information.
  - Guarantees integrity and prevents tampering.
  - Authenticates the server identity via the SSL certificate.

---

### 4) Firewalls — Purpose
- Firewalls enforce **network segmentation** and the **principle of least privilege**:
  - **Firewall #1:** Only allows internet → load balancer traffic on ports 80/443.
  - **Firewall #2:** Only allows web servers to reach the application servers.
  - **Firewall #3:** Only allows app servers to connect to the database (port 3306).

---

### 5) Monitoring
- **Why monitoring is important:** Detects outages, performance issues, and security anomalies.
- **How monitoring works:** A monitoring client runs on each server and sends logs, metrics, and traces to a central monitoring system.
- **To monitor QPS (Queries Per Second):**
  1. Enable Nginx or HAProxy metrics.
  2. Collect request counts regularly.
  3. Compute QPS and display it on dashboards.
  4. Configure alerts when QPS exceeds capacity.

---

### 6) Why We Added Each Component
- **3 Firewalls:** Add security between public, application, and database layers.
- **SSL certificate:** Enables HTTPS to secure user traffic.
- **Monitoring clients:** Provide visibility into server health and performance.

---

### 7) Infrastructure Issues
- **SSL terminated only at the load balancer:**
  - Traffic between load balancer and backend servers is unencrypted if no end-to-end TLS is used.
- **Only one MySQL Primary:**
  - This is a Single Point of Failure (SPOF) for writes. If it fails, no writes are possible until failover.
- **Servers with identical components:**
  - Increases risk: a misconfiguration or attack on one server could affect all.
  - Encourages storing state locally (sessions, files), making scaling and failover harder.

---
