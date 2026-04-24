# DevOps-Assignment
CloudEagle DevOps Assignment



**================= DEVOPS FLOW =================**

Developer → GitHub → Jenkins → GCR → MIG (Blue/Green Deploy)

**================= OBSERVABILITY =================**

sync-service → Cloud Logging → Cloud Monitoring → Alerts

**================= SECURITY =================**

IAM Roles:
- Jenkins → Compute + GCR + Secrets
- VM → Secret Manager (read-only)
