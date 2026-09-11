# Cloud Run mapping note

This pipeline currently pushes to Docker Hub and deploys locally via
`docker compose up -d`. Below is how the same stages map onto Google Cloud
Run instead. Documentation only — nothing here is deployed.

## Push stage: Docker Hub -> Artifact Registry

Replace the `docker/login-action` + `build-push-action` steps' target
registry. Instead of Docker Hub:

```yaml
- name: Auth to Google Cloud
  uses: google-github-actions/auth@v2
  with:
    credentials_json: ${{ secrets.GCP_SA_KEY }}

- name: Configure Docker for Artifact Registry
  run: gcloud auth configure-docker ${{ vars.GCP_REGION }}-docker.pkg.dev

- name: Build and push image
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ${{ vars.GCP_REGION }}-docker.pkg.dev/${{ vars.GCP_PROJECT }}/${{ vars.GCP_REPO }}/app:${{ github.sha }}
```

## Deploy stage: docker compose -> gcloud run deploy

```yaml
- name: Deploy to Cloud Run
  run: |
    gcloud run deploy app \
      --image ${{ vars.GCP_REGION }}-docker.pkg.dev/${{ vars.GCP_PROJECT }}/${{ vars.GCP_REPO }}/app:${{ github.sha }} \
      --region ${{ vars.GCP_REGION }} \
      --platform managed \
      --port 8080 \
      --allow-unauthenticated
```

Still gated by `needs: build`, same as the local compose deploy.

## Auth

- A GCP service account with `roles/artifactregistry.writer` and
  `roles/run.admin` (plus `roles/iam.serviceAccountUser` on the runtime SA).
- Its JSON key stored as a GitHub Actions secret `GCP_SA_KEY` (or, better,
  workload identity federation instead of a long-lived key — no secret to
  rotate).
- Project/region/repo names as repo variables (`GCP_PROJECT`, `GCP_REGION`,
  `GCP_REPO`) rather than hardcoded.

## Health check

After `gcloud run deploy`, the command prints the service URL. The same
`curl -f <url>/health` check used locally applies there, just against the
Cloud Run URL instead of `localhost:8080`.
