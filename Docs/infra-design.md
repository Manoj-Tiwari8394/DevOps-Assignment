## Compute choice (GKE / Compute Engine / Cloud Run)

### Option comparison

| Option           | Pros                                                                 | Cons                                                          |
|------------------|----------------------------------------------------------------------|----------------------------------------------------------------|
| **GKE**          | Auto‑scaling, rolling updates, ideal for microservices.           | Steeper learning curve and higher operational overhead.     |
| **Compute Engine** | Simple, full control, easy to run Spring Boot on VMs.           | Manual scaling, no built‑in rolling update mechanism.       |
| **Cloud Run**    | Fully managed, pay‑per‑use, auto‑scaling.                         | Not ideal for long‑running services with persistent connections. |

Given the requirements:

- The service already runs on **GCP VMs**.
- Startup constraints (cost, simplicity).
- Need for auto‑scaling and reasonable control.

### Recommendation

We recommend **GCE VMs inside a Managed Instance Group (MIG)** with:

- Minimum 2 VMs for production.
- Auto‑scaling based on CPU and HTTP request rate.
- Health checks via Spring Boot’s `/actuator/health` endpoint.

This approach balances automation, cost, and control. Later, if the team grows, we can migrate to GKE.

---

## MongoDB hosting approach

We propose using **MongoDB Atlas** for QA, staging, and production:

- Fully managed service with high availability and automated backups.
- No need for internal DB operations team.
- Easy to scale and monitor.
- Decoupled from GCP VMs, allowing independent upgrades.

Connection details (URI, username, password) are stored in **GCP Secret Manager** and injected into the Spring Boot app at runtime.

If Atlas is not feasible due to cost, an alternative is:

- A MongoDB replica set on GCP VMs (3 nodes: primary + 2 secondaries).
- Or a StatefulSet on GKE if using Kubernetes later.

---

## Networking basics (VPC, ingress)

### VPC and subnets

- Create a **custom VPC** for the `sync-service` project.
- Subnets:
  - `sync-service-subnet` – private subnet for the Spring Boot VMs.
  - `mgmt-subnet` – optional management subnet for bastion or CI tools.
- Use **Firewall rules** to restrict traffic:

  - Allow HTTP/HTTPS from the GCP HTTP(S) Load Balancer to the VMs on port 8080.
  - Allow MongoDB Atlas (or MongoDB replica set) to connect to the VMs on port 27017 (if required).
  - Block all other unnecessary traffic.

### Ingress and load balancing

- Use **GCP HTTP(S) Load Balancer**:
  - Public IP for external users.
  - HTTPS with SSL certificate (GCP managed certificate).
- Configure health checks on the Spring Boot app’s `/actuator/health` endpoint.
- Route traffic to the backend instance group(s) behind the load balancer.

### Admin access

- No direct SSH from the internet.
- Admin access via:
  - **bastion host** in a management subnet.

---

## Secrets & IAM

### Secrets management

- Store all secrets (MongoDB connection strings, API keys, etc.) in **GCP Secret Manager**.
- The Jenkins pipeline and deployment scripts:
  - Authenticate with a GCP service account.
  - Fetch secrets at deployment time.
  - Inject them into environment variables or temporary files.
- The Spring Boot app reads secrets from environment variables at startup.

This keeps secrets out of Git and leverages GCP’s IAM for access control.

### IAM roles

- **Jenkins service account**:
  - `roles/storage.admin` – to push images to GCR.
  - `roles/compute.admin` – to manage VMs and instance groups.
  - `roles/secretmanager.viewer` – to read secrets.
- **`sync-service` VM service account**:
  - `roles/secretmanager.viewer` – to read its own secrets.
  - No broad admin roles.

We follow the principle of least privilege so that no component has more permissions than strictly needed.

---

## Logging & monitoring stack

### Logging

- The Spring Boot app writes logs to `stdout` and `stderr` in JSON format.
- GCP **Cloud Logging** automatically collects logs from VMs and containers.
- Use structured JSON logging for easier filtering and alerting (e.g., `{"level":"ERROR","endpoint":"/api/sync"}`).

### Monitoring

- Use **GCP Cloud Monitoring** to monitor:
  - CPU, memory, and disk usage of VMs.
  - HTTP request rate, latency, and error rate.
- Add custom metrics via Spring Boot Actuator’s `/actuator/prometheus` endpoint if needed.
- Optionally, later integrate **Prometheus + Grafana** for dashboards.

### Alerting

- Create alerts for:
  - HTTP 5xx errors above a threshold.
  - High latency or slow endpoints.
  - Instance down or health check failures.
- Configure notifications via email or Slack.

This setup ensures that the service is observable, maintainable, and resilient under production load.






**================= DEVOPS FLOW =================**

Developer → GitHub → Jenkins → GCR → MIG (Blue/Green Deploy)

**================= OBSERVABILITY =================**

sync-service → Cloud Logging → Cloud Monitoring → Alerts

**================= SECURITY =================**

IAM Roles:
- Jenkins → Compute + GCR + Secrets
- VM → Secret Manager (read-only)
