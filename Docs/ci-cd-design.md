## Branching strategy and environment mapping

We follow a Gitflow‑inspired branching model with the following branches:

- `main` (or `master`)  
  Stable branch that represents the production‑ready state.  
  Direct push is disabled; changes must be merged via Pull Requests (PRs).

- `develop`  
  Integration branch for all features.  
  New work is merged into `develop` after PR review.

- `feature/*`  
  Feature branches such as `feature/user-sync`, `feature/batch-job`, etc.  
  Always created from `develop` and merged back into `develop` via PR.

- `release/*`  
  Short‑lived release branches cut from `develop` for final QA and staging tests.  
  After sign‑off, `release/*` is merged into `main` and tagged.

- `hotfix/*`  
  Hotfix branches created from `main` for urgent production fixes.  
  After testing, they are merged into both `main` and `develop`.

### Mapping branches to environments

- `feature/*` PRs  
  Trigger builds and optional ephemeral deployments to a QA environment for integration tests.  
  No permanent deployment to QA unless explicitly allowed.

- `develop` merges  
  Automatically build, test, and deploy to the QA environment.

- `release/*` merges  
  Automatically build, test, and deploy to the staging environment.

- Tags on `main`  
  Represent production releases.  
  Deployment to `prod` requires manual approval.

This mapping ensures that:

- QA always runs the latest integrations.
- Staging mirrors the exact code that will go to production.
- Production only receives tagged, approved releases.

### Preventing accidental production deployments

- Only **tagged commits** on `main` are allowed to reach production.
- The Jenkins pipeline has separate stages for `qa`, `staging`, and `prod`; prod deployment is gated by a manual approval step.
- Pull requests targeting `main` are disabled; only `develop` and `release/*` can be merged via PR.
- Jenkins uses separate GCP service accounts:
  - One for QA and staging (with relevant compute and storage permissions).
  - One for production (with the minimal permissions required plus explicit approval).
- Branch‑based triggers:
  - `develop` → deploys to QA.
  - `release/*` → deploys to staging.
  - Tags on `main` → deploys to production (only after manual approval).

---

## Jenkins pipeline design

### High‑level pipeline stages

The Jenkins pipeline is declarative and follows these stages:

1. **Checkout**  
   - Checkout the Git repository.  
   - Extract the branch name and determine the target environment (`qa`, `staging`, `prod`, or `pr`).

2. **Build**  
   - Run `mvn clean package` (or `./mvnw`) to build the Spring Boot JAR.  
   - Skip tests in this stage for speed; tests run in a separate stage.

3. **Test**  
   - Run unit and integration tests.  
   - Archive JUnit test results if needed.

4. **Code Quality**  
   - Run static analysis (SonarQube / Checkstyle or equivalent).  
   - Fail the build if quality gates are not met.

5. **Build Docker Image**  
   - Build a Docker image with environment‑specific tags:
     - `qa-${BUILD_NUMBER}` for QA.
     - `staging-${GIT_COMMIT}` for staging.
     - `prod-${TAG}` for production (if using Git tags).

6. **Push Image**  
   - Authenticate to GCR (Google Container Registry).  
   - Push the image to the respective environment tag.

7. **Deploy**  
   - Deploy to the target environment:
     - QA: automatic deployment to GCP VMs or container infrastructure.
     - Staging: automatic deployment after tagging.
     - Production: manual approval then deployment.

8. **Rollback on failure (prod)**  
   - If the production deployment fails, trigger a rollback script that reverts the load balancer to the previous healthy backend.

9. **Notifications**  
   - Send notifications on success or failure (Slack / email).

### PR vs merged behavior

#### Pull Request (feature branch)

- Triggered when a PR is opened or updated.
- Pipeline stages:
  - Checkout
  - Build
  - Test
  - Code Quality
- No deployment to permanent environments unless explicitly enabled (ephemeral QA only).
- If any stage fails, the PR build fails and the change is blocked.

#### Merge to `develop`

- Triggered when a PR is merged into `develop`.
- Pipeline stages:
  - Checkout
  - Build
  - Test
  - Code Quality
  - Build Docker image with `qa-${BUILD_NUMBER}` tag.
  - Push image to GCR.
  - Deploy to QA environment.
  - Run smoke tests.
- Notify team on failure.

#### Merge to `release/*` or `main` (tag)

- Triggered when a PR is merged into `release/*` or when a tag is created on `main`.
- Pipeline stages:
  - Same as above, but images are tagged for `staging` or `prod`.
  - Deploy to staging automatically.
  - Deploy to production after manual approval.

---

## Rollback strategy if deployment fails

If deployment to production fails:

- **Blue/Green rollback on VMs**  
  - We maintain two instance groups: `prod-blue` and `prod-green`.  
  - Each release targets one instance group while the other remains as the previous version.  
  - If the new release fails health checks or causes errors, the GCP HTTP(S) Load Balancer is updated to route traffic back to the previous healthy instance group.  
  - The failed image is retained for post‑mortem but not for production traffic.

- **Kubernetes‑style rollback (future)**  
  - If the service is later migrated to GKE, we can use `kubectl rollout undo` to revert to the previous deployment revision.  
  - Health checks and monitoring will detect issues early so rollback can be triggered automatically if needed.

---

## Configuration management

### Environment‑specific configuration

We use Spring Boot’s profile‑based configuration:

- `src/main/resources/application.yml`  
  Common configuration shared across environments.

- `src/main/resources/application-qa.yml`  
  QA‑specific properties (DB URL, feature flags, etc.).

- `src/main/resources/application-staging.yml`  
  Staging‑specific properties.

- `src/main/resources/application-prod.yml`  
  Production‑specific properties.

During startup, we set the Spring profile based on the environment:

```bash
java -Dspring.profiles.active=qa -jar sync-service.jar
```

This ensures that the correct environment‑specific configuration is loaded without code changes.

### Secrets handling (MongoDB, API keys)

- Store MongoDB connection strings, API keys, and other secrets in **GCP Secret Manager**.
- The Jenkins pipeline and deployment script:
  - Fetch secrets using a GCP service account.
  - Inject them into the environment (e.g., as environment variables or temporary files) at deploy time.
- Never commit secrets to Git or store them in plain text configuration files.
- The Spring Boot app reads secrets from environment variables at runtime (for example, `MONGO_USERNAME`, `MONGO_PASSWORD`, etc.).

This approach keeps sensitive data out of version control and integrates well with GCP IAM.

---

## Deployment strategy (Blue/Green vs Rolling vs Recreate)

We choose **Blue/Green deployment** for the following reasons:

- **Zero or minimal downtime**  
  - Traffic is switched instantly from the old backend to the new backend via the GCP HTTP(S) Load Balancer.
- **Fast rollback**  
  - If the new version fails, traffic can be reverted to the previous healthy instance group in seconds.
- **Simpler testing**  
  - The new version can be tested in parallel with the old version before switching traffic.
- **Operational clarity**  
  - Each release runs on its own set of VMs, which simplifies debugging and rollback.

### Why not Rolling or Recreate?

- **Recreate**  
  - Involves stopping all old instances before starting new ones, which causes downtime and is not acceptable for production.
- **Rolling update**  
  - Works well on Kubernetes but is more complex to orchestrate on raw GCP VMs without managed groups and built‑in rolling logic.

### Minimal‑downtime approach

- Use a **GCP HTTP(S) Load Balancer** in front of two Managed Instance Groups:
  - `prod-blue` (current version)
  - `prod-green` (new version)
- During deployment:
  - Prepare a new instance group with the updated Docker image.
  - Wait for health checks to pass.
  - Update the backend configuration of the load balancer to point to the new instance group.
- Rollback:
  - If the new version fails, update the backend to point back to the previous healthy instance group.
- This setup provides zero‑downtime deployments and fast rollback, suitable for a startup‑constrained environment.
