# Cloud Mapping: Local Health Checks to GCP Cloud Run & Kubernetes Probes

This document maps local Docker Compose health checks to managed cloud environments (Google Cloud Run and Kubernetes).

---

## Overview of Health Checks

| Check Type | Endpoint | Purpose | Scope | Action on Failure |
| :--- | :--- | :--- | :--- | :--- |
| **Liveness Probe** | `GET /health` | Shallow check to verify process is alive | Process runtime only | Restart container instance |
| **Readiness Probe** | `GET /ready` | Deep check to verify downstream dependencies (e.g., PostgreSQL DB) | Application + Dependencies | Remove instance from load balancer routing (do not kill container) |
| **Startup Probe** | `GET /ready` or `/health` | Determines if app initialization complete | Container startup | Delay liveness/readiness checks until passing |

---

## 1. Local Docker Compose Mapping

In `docker-compose.yml`:
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3000/ready"]
  interval: 10s
  timeout: 3s
  start_period: 20s
  retries: 3
restart: unless-stopped
```
- **Behavior**: Periodically queries `GET /ready`. If it fails 3 consecutive times, container status changes to `unhealthy`. The `restart: unless-stopped` policy tells Docker to restart the container if it exits or fails runtime checks.

---

## 2. GCP Cloud Run Mapping

Google Cloud Run supports **Liveness Probes** and **Startup Probes**:

### A. Liveness Probe (`GET /health`)
- **Configuration**:
  - HTTP GET path: `/health`
  - Port: `3000` (or `PORT` environment variable)
  - Initial delay: `5s`
  - Timeout: `3s`
  - Period: `10s`
  - Failure threshold: `3`
- **Purpose**: Verifies that the Node.js process is responsive and not deadlocked. If it fails, Cloud Run stops and replaces the unhealthy instance.
- **Why `/health` instead of `/ready` for Liveness**: If the database experiences temporary downtime, killing and restarting the API container will not fix the database. Using `/health` prevents Cloud Run from entering an infinite container restart loop during external dependency outages.

### B. Startup Probe (`GET /ready` or `GET /health`)
- **Configuration**:
  - HTTP GET path: `/ready`
  - Initial delay: `0s`
  - Period: `5s`
  - Failure threshold: `6`
- **Purpose**: Gives container extra time during cold starts or initial database connection setup. Traffic is not sent to the container until the startup probe succeeds.

---

## 3. Kubernetes Probes Mapping

In a Kubernetes deployment (e.g. GKE), probes map as follows:

```yaml
spec:
  containers:
    - name: orders-api
      image: orders-api:latest
      ports:
        - containerPort: 3000
      livenessProbe:
        httpGet:
          path: /health
          port: 3000
        initialDelaySeconds: 10
        periodSeconds: 10
        timeoutSeconds: 3
        failureThreshold: 3
      readinessProbe:
        httpGet:
          path: /ready
          port: 3000
        initialDelaySeconds: 5
        periodSeconds: 5
        timeoutSeconds: 3
        failureThreshold: 3
```

- **Readiness Probe (`GET /ready`)**: If PostgreSQL goes down, `readinessProbe` fails (`503 NOT READY`). Kubernetes temporarily removes the Pod IP from the Service endpoints without killing the Pod. No client requests are routed to the Pod until PostgreSQL recovers.
- **Liveness Probe (`GET /health`)**: Remains `200 OK` as long as Node.js process functions. If Node.js freezes/crashes, `livenessProbe` fails and Kubernetes restarts the Pod.

---

## Key Design Architectural Principles

1. **Decouple Process Liveness from Dependency Readiness**:
   - Liveness (`/health`) answers: *"Is the process running?"*
   - Readiness (`/ready`) answers: *"Can the service handle incoming traffic?"*
2. **Prevent Cascading Restart Storms**:
   - Marking containers dead (liveness failure) when a database is temporarily unreachable worsens outage recovery.
   - Using `/ready` for load balancer traffic routing protects users while allowing dependencies to recover gracefully.
