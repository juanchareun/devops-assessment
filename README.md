# DevOps Assessment

Small .NET 8 API used to build a CI/CD pipeline with Docker, GitHub Actions, Trivy, and GitHub Container Registry.

## Endpoints

### Health

```text
GET /health
```

Returns a basic health response.

### Version

```text
GET /version
```

Returns the value of `APP_VERSION`, or `dev` if it is not set.

## Run locally

```bash
cd DevOpsAssessment
dotnet run
```

Test:

```bash
curl http://localhost:5205/health
curl http://localhost:5205/version
```

To test a version:

```bash
APP_VERSION=1.0.0 dotnet run
```

## Docker

Build:

```bash
docker build -t devops-assessment ./DevOpsAssessment
```

Run:

```bash
docker run --rm \
  -p 8080:8080 \
  -e APP_VERSION=local \
  devops-assessment
```

Test:

```bash
curl http://localhost:8080/health
curl http://localhost:8080/version
```

## CI/CD

The GitHub Actions workflow handles:

```text
format
→ build/test
→ dependency scan
→ Docker build
→ Trivy scan
→ GHCR
→ deploy
```

The Trivy image scan acts as the security gate before publishing and deployment.

Manual redeployment is also available through `workflow_dispatch`.

See [PIPELINE.md](PIPELINE.md) for the workflow diagram.

## Notes

Implementation notes, troubleshooting, and assessment questions are documented in [WORKLOG.md](WORKLOG.md).
