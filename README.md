# aegis-backend

The Flask/Gunicorn API uses PostgreSQL through `DATABASE_URL` and listens on port 5000.

| Endpoint | Purpose |
| --- | --- |
| `GET /api/health` | Liveness |
| `GET /api/ready` | Database readiness |
| `GET, POST /api/todos` | List or create todos |
| `PATCH, DELETE /api/todos/<id>` | Toggle or delete a todo |

`app.py`, `requirements.txt` and `Dockerfile` live at the repository root.

## Related repositories

- [aegis-app-manifests](https://github.com/oxiys/aegis-app-manifests): Kubernetes resources, policies, GitOps image tags and local Compose orchestration.
- [aegis-backend](https://github.com/oxiys/aegis-backend): Flask API and its image pipeline.
- [aegis-frontend](https://github.com/oxiys/aegis-frontend): Nginx UI and its image pipeline.

## Local development

Place the three repositories next to one another, then run:

```bash
cd ../aegis-app-manifests
docker compose up --build
```

Open http://localhost:3000. PostgreSQL data is kept in the `pgdata` volume.
The example database credentials are for the mock application.

Build this repository independently with:

```bash
docker build -t oxiys/backend:local .
```

## CI/CD setup

Pull requests and pushes to `main` scan secrets, source, Dockerfile and the built
image. Builds use the repository root. Releases additionally push the scanned
image as `oxiys/backend:<commit-sha>` and `:latest`, then update only
`k8s/08-backend-deployment.yaml` in `oxiys/aegis-app-manifests`.

Set these repository Actions secrets:

| Secret | Value |
| --- | --- |
| `DOCKER_USERNAME` | Docker Hub account with write access to `oxiys/backend` |
| `DOCKER_PASSWORD` | Docker Hub access token |
| `GITOPS_TOKEN` | Fine-grained GitHub token restricted to `oxiys/aegis-app-manifests`, with Contents read/write |

The normal `GITHUB_TOKEN` is scoped to its own repository; a separate token is
needed to update the manifests. See [GitHub authentication documentation](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token).
The token's identity must also be allowed to push to the manifests' `main` branch.
This preserves the original direct-push GitOps behavior; a branch requiring PRs
needs a PR-based update workflow before enabling releases.

After merging the manifest migration and configuring all three secrets, set the
Actions variable `RELEASE_ENABLED` to `true`, then run the `Aegis Backend`
workflow manually once. Until then, builds and security scans run without publishing.
The manifest validation workflow runs on the cross-repository update; commits no
longer use `[skip ci]`. Frontend and backend push collisions are retried with rebase
and ordinary, non-forced pushes.

The Docker Hub image names and Kubernetes paths remain the same as before the split.
Git history was filtered from the original `backend/` directory, retaining
relevant authors, dates and commit messages; the resulting commit SHAs change.
