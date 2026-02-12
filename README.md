# Coolify Deploy Actions

Reusable GitHub Actions for deploying applications on [Coolify](https://coolify.io) via its API. Designed for workflows where you version deployments by setting an environment variable (e.g., a Docker image tag) and then triggering a redeploy.

## Actions

### `naffiq/coolify-deploy-actions/tag`

Updates an environment variable on a Coolify application via `PATCH /api/v1/applications/{uuid}/envs`. Use this to set the Docker image tag (or any other env var) before deploying.

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `coolify_url` | yes | | Coolify instance base URL |
| `coolify_token` | yes | | Coolify API bearer token |
| `app_uuid` | yes | | Application UUID |
| `key` | yes | | Env var key (e.g., `IMAGE_TAG`) |
| `value` | yes | | Env var value (e.g., `v1.2.3`) |
| `is_preview` | no | `false` | Whether this is a preview env var |

### `naffiq/coolify-deploy-actions/deploy`

Triggers a deployment on Coolify via `GET /api/v1/deploy`. At least one of `uuid` or `tag` must be provided.

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `coolify_url` | yes | | Coolify instance base URL |
| `coolify_token` | yes | | Coolify API bearer token |
| `uuid` | no | | Resource UUID(s), comma-separated |
| `tag` | no | | Coolify tag name(s), comma-separated |
| `force` | no | `false` | Force rebuild |

Both actions expose a `response` output containing the raw API response body.

## Versioned Deployments via Environment Variables

Instead of building images directly on Coolify, you can build and push Docker images in CI and use an environment variable to control which version Coolify deploys. This gives you full control over the image lifecycle while letting Coolify handle orchestration.

### Setup

1. **In your Coolify application**, use an environment variable to reference the image tag in your Docker Compose or deployment config:

   ```yaml
   # docker-compose.yml deployed by Coolify
   services:
     app:
       image: ghcr.io/your-org/your-app:${IMAGE_TAG:-latest}
   ```

2. **Add the `IMAGE_TAG` env var** to your application in the Coolify dashboard (or via the API). Set it to `latest` or any initial value.

3. **Store your Coolify credentials** as GitHub repository secrets:
   - `COOLIFY_URL` — your Coolify instance URL (e.g., `https://coolify.example.com`)
   - `COOLIFY_TOKEN` — an API token from Coolify (Settings → API Tokens)
   - `COOLIFY_APP_UUID` — the UUID of your application (visible in the Coolify dashboard URL)

### Example Workflow

This workflow builds a Docker image on push to `main`, tags it with the commit SHA, pushes it to GHCR, then updates the image tag on Coolify and triggers a deployment:

```yaml
name: Build & Deploy

on:
  push:
    branches: [main]

env:
  IMAGE: ghcr.io/${{ github.repository }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: |
            ${{ env.IMAGE }}:${{ github.sha }}
            ${{ env.IMAGE }}:latest

      # Set the IMAGE_TAG env var on Coolify to the new commit SHA
      - uses: naffiq/coolify-deploy-actions/tag@main
        with:
          coolify_url: ${{ secrets.COOLIFY_URL }}
          coolify_token: ${{ secrets.COOLIFY_TOKEN }}
          app_uuid: ${{ secrets.COOLIFY_APP_UUID }}
          key: IMAGE_TAG
          value: ${{ github.sha }}

      # Trigger the deployment
      - uses: naffiq/coolify-deploy-actions/deploy@main
        with:
          coolify_url: ${{ secrets.COOLIFY_URL }}
          coolify_token: ${{ secrets.COOLIFY_TOKEN }}
          uuid: ${{ secrets.COOLIFY_APP_UUID }}
```

### Deploying Multiple Services by Tag

If you have multiple Coolify resources grouped under a tag, you can update each app's image version individually and then deploy them all at once:

```yaml
      # Update image tags for each service
      - uses: naffiq/coolify-deploy-actions/tag@main
        with:
          coolify_url: ${{ secrets.COOLIFY_URL }}
          coolify_token: ${{ secrets.COOLIFY_TOKEN }}
          app_uuid: ${{ secrets.API_UUID }}
          key: IMAGE_TAG
          value: ${{ github.sha }}

      - uses: naffiq/coolify-deploy-actions/tag@main
        with:
          coolify_url: ${{ secrets.COOLIFY_URL }}
          coolify_token: ${{ secrets.COOLIFY_TOKEN }}
          app_uuid: ${{ secrets.WORKER_UUID }}
          key: IMAGE_TAG
          value: ${{ github.sha }}

      # Deploy all services under the "production" tag at once
      - uses: naffiq/coolify-deploy-actions/deploy@main
        with:
          coolify_url: ${{ secrets.COOLIFY_URL }}
          coolify_token: ${{ secrets.COOLIFY_TOKEN }}
          tag: production
```

### Using Semantic Version Tags

You can also use release tags instead of commit SHAs for cleaner versioning:

```yaml
on:
  release:
    types: [published]

# ...

      - uses: naffiq/coolify-deploy-actions/tag@main
        with:
          coolify_url: ${{ secrets.COOLIFY_URL }}
          coolify_token: ${{ secrets.COOLIFY_TOKEN }}
          app_uuid: ${{ secrets.COOLIFY_APP_UUID }}
          key: IMAGE_TAG
          value: ${{ github.event.release.tag_name }}
```

## API Token

Generate an API token in your Coolify instance under **Settings → API Tokens**. The token needs permission to manage applications and trigger deployments. Store it as a GitHub Actions secret — the actions automatically mask it from workflow logs.
