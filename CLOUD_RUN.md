# Mapping the CI/CD Pipeline to Google Cloud Run

This document maps our local **Docker Hub + Docker Compose** GitHub Actions pipeline to a production **Google Artifact Registry + Google Cloud Run** deployment pipeline.

---

## 1. Stage-by-Stage Architecture Comparison

| Pipeline Stage | Current Pipeline (Docker Hub + Compose) | Cloud Run Equivalent (GCP) |
| :--- | :--- | :--- |
| **Container Registry** | Docker Hub (`docker.io/<user>/app:<sha>`) | Google Artifact Registry (`<REGION>-docker.pkg.dev/<PROJECT_ID>/<REPO>/app:<sha>`) |
| **Authentication** | `docker/login-action@v3` using `DOCKERHUB_USER` and `DOCKERHUB_TOKEN` secrets | `google-github-actions/auth@v2` via Workload Identity Federation (or GCP Service Account Key) + `gcloud auth configure-docker <REGION>-docker.pkg.dev` |
| **Build & Push** | `docker/build-push-action@v5` pushing `<DOCKERHUB_USER>/app:${{ github.sha }}` | `docker/build-push-action@v5` pushing `<REGION>-docker.pkg.dev/<PROJECT_ID>/<REPO>/app:${{ github.sha }}` |
| **Deployment** | `IMAGE=<user>/app:${{ github.sha }} docker compose up -d` on the runner VM | `gcloud run deploy app --image <REGION>-docker.pkg.dev/<PROJECT_ID>/<REPO>/app:${{ github.sha }} --region <REGION>` (or `google-github-actions/deploy-cloudrun@v2`) |
| **Health Verification** | `curl -i -f http://localhost:8080/health` (HTTP 200) | `curl -i -f <CLOUD_RUN_SERVICE_URL>/health` (HTTP 200) |

---

## 2. Detailed Component Mapping

### Registry: Docker Hub → Google Artifact Registry
- **Current:** Images are tagged as `${{ secrets.DOCKERHUB_USER }}/app:${{ github.sha }}` and pushed to Docker Hub.
- **Cloud Run:** Images are stored in a regional Google Artifact Registry Docker repository using the immutable commit SHA tag:
  ```text
  REGION-docker.pkg.dev/PROJECT_ID/REPOSITORY_NAME/app:${{ github.sha }}
  ```

### Authentication: `docker/login-action` → `google-github-actions/auth`
- **Current:** Uses static Docker Hub Personal Access Tokens (`DOCKERHUB_USER`, `DOCKERHUB_TOKEN`) with `docker/login-action@v3`.
- **Cloud Run:** Uses `google-github-actions/auth@v2` with **Workload Identity Federation** (recommended, keyless OIDC authentication) or a GCP Service Account JSON key (`GCP_SA_KEY`), followed by configuring Docker for Artifact Registry:
  ```yaml
  - name: Authenticate to Google Cloud
    uses: google-github-actions/auth@v2
    with:
      workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
      service_account: ${{ secrets.WIF_SERVICE_ACCOUNT }}

  - name: Set up Cloud SDK
    uses: google-github-actions/setup-gcloud@v2

  - name: Configure Docker for Artifact Registry
    run: gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev --quiet
  ```

### Deploy: `docker compose up -d` → `gcloud run deploy`
- **Current:** Starts the container locally on the GitHub Actions runner via `docker compose up -d` using the `IMAGE` environment variable.
- **Cloud Run:** Deploys a new immutable revision to Google Cloud Run listening on port `8080`:
  ```yaml
  - name: Deploy to Cloud Run
    run: |
      gcloud run deploy app \
        --image ${{ env.REGION }}-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/app-repo/app:${{ github.sha }} \
        --region ${{ env.REGION }} \
        --platform managed \
        --port 8080 \
        --allow-unauthenticated
  ```

### Verification: `localhost:8080/health` → `<CLOUD_RUN_SERVICE_URL>/health`
- **Current:** Queries the local container port directly with `curl -i -f http://localhost:8080/health`.
- **Cloud Run:** Retrieves the live HTTPS endpoint URL assigned by Cloud Run and verifies that `/health` returns `HTTP/2 200` and `{"status":"ok"}`:
  ```yaml
  - name: Verify Cloud Run Health Endpoint
    run: |
      SERVICE_URL=$(gcloud run services describe app --region ${{ env.REGION }} --format 'value(status.url)')
      curl -i -f "${SERVICE_URL}/health"
  ```
