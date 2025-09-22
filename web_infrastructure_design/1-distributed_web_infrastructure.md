# 1. Distributed Web Infrastructure

A three-server web infrastructure for **www.foobar.com** with:
- 1 load balancer (HAProxy)
- 2 backend servers (each: Nginx, application server, codebase, MySQL with Primary/Replica topology)

---

## 🌐 Diagram – Flowchart (Mermaid)

```mermaid
flowchart TB
    U["User (Browser)"] -->|DNS resolves www.foobar.com| LB["HAProxy Load Balancer"]

    subgraph S1["Server #1"]
      N1["Nginx (web)"] --> A1["App server (code)"]
      A1 --> D1[("MySQL - Primary")]
    end

    subgraph S2["Server #2"]
      N2["Nginx (web)"] --> A2["App server (code)"]
      A2 --> D2[("MySQL - Replica")]
    end

    LB -->|HTTP/HTTPS| N1
    LB -->|HTTP/HTTPS| N2

    D1 <-->|Async replication| D2
```
```mermaid
sequenceDiagram
    participant User as User (Browser)
    participant DNS as DNS
    participant LB as HAProxy
    participant N1 as Nginx #1
    participant A1 as App #1
    participant D1 as MySQL Primary
    participant N2 as Nginx #2
    participant A2 as App #2
    participant D2 as MySQL Replica

    User->>DNS: Resolve www.foobar.com
    DNS-->>User: IP of LB
    User->>LB: HTTPS GET /
    alt LB routes to Server #1 (e.g., Round-Robin)
        LB->>N1: Forward request
        N1->>A1: Proxy dynamic request
        A1->>D1: Write/critical read
        D1-->>A1: Result
        A1-->>N1: HTML/JSON
        N1-->>User: 200 OK
    else LB routes to Server #2
        LB->>N2: Forward request
        N2->>A2: Proxy dynamic request
        opt Read-only workload
            A2->>D2: Read from replica
            D2-->>A2: Result
        end
        A2-->>N2: HTML/JSON
        N2-->>User: 200 OK
    end
```
---

## 📄 Explanation

### 1) User Request and DNS Resolution
1. The user opens a browser and types `www.foobar.com`.
2. The browser asks the DNS to resolve the domain name into an IP address.
3. The DNS replies with the IP address of the **load balancer**.
4. The browser sends an HTTPS/HTTP request to the load balancer.

---

### 2) Load Balancer (HAProxy)
1. The load balancer receives the request.
2. It uses a **distribution algorithm** (for example, Round-Robin) to choose a backend server.
3. It forwards the request to one of the two Nginx web servers.

---

### 3) Backend Servers
1. **Nginx (web server):**
   - Serves static files (HTML, CSS, JS) directly.
   - For dynamic requests, it forwards them to the application server.
2. **Application server:**
   - Runs the business logic (your code).
   - If data is needed, it queries the database.
3. **MySQL Primary–Replica:**
   - The **Primary** database handles all **writes** and critical reads.
   - The **Replica** receives asynchronous updates from the Primary and can be used for **read-only queries**.

---

### 4) Why We Added Each Component
- **Load balancer:** Distributes traffic between the two servers to increase availability and reliability.
- **Two servers:** Provide redundancy — if one server fails, the other continues to serve users.
- **Primary–Replica database setup:** Improves read performance by offloading SELECT queries to the replica.

---

### 5) Infrastructure Issues
- **Single Point of Failure (SPOF):**
  - The load balancer is a SPOF — if it goes down, no traffic can reach the servers.
  - The MySQL Primary is also a SPOF — if it fails, no writes are possible until manual failover.
- **Security issues:** No firewall and no HTTPS termination are configured.
- **No monitoring:** There is no way to detect failures or measure performance.

---
