# DevOps Assessment Worklog

## Environment

Environment used for the assessment:

- Ubuntu Linux VM
- .NET 8 SDK
- Docker
- Trivy
- Git
- GitHub
- GitHub Container Registry (GHCR)

---

# Part 1 — Scaffold a Real Service

Created an ASP.NET Core Web API using .NET 8.

The service contains two endpoints:

- `GET /health`
- `GET /version`

`/health` returns HTTP 200 with a small JSON response.

`/version` reads the `APP_VERSION` environment variable and defaults to `dev` if the variable is not set.

### Local Testing

Started the service locally with:

```bash
dotnet run
```

Verified the health endpoint:

```bash
curl http://localhost:5205/health
```

Verified the default version:

```bash
curl http://localhost:5205/version
```

The response returned:

```text
dev
```

Then started the application with an environment variable:

```bash
APP_VERSION=1.0.0 dotnet run
```

Verified `/version` again and confirmed it returned:

```text
1.0.0
```

This confirmed that both endpoints worked and that `APP_VERSION` could override the default value.

---

# Part 2 — Vulnerability Scanning

## Dependency Baseline

First ran:

```bash
dotnet list package --vulnerable --include-transitive
```

No vulnerable packages were initially reported.

## Deliberately Vulnerable Dependency

Added an intentionally vulnerable version of Newtonsoft.Json:

```bash
dotnet add package Newtonsoft.Json --version 9.0.1
```

Ran the dependency scan again:

```bash
dotnet list package --vulnerable --include-transitive
```

The scan reported a HIGH severity vulnerability in Newtonsoft.Json.

The vulnerable version was:

```text
Newtonsoft.Json 9.0.1
```

The finding included:

```text
CVE-2024-21907
```

## Dockerfile

Created a multi-stage Dockerfile using:

- .NET 8 SDK as the build stage
- ASP.NET 8 runtime as the final stage
- Release publishing
- framework-dependent deployment
- a non-root `appuser`
- port `8080`
- Docker `HEALTHCHECK`

The application is copied from the build stage into the smaller runtime image.

The final container runs as:

```text
appuser
```

instead of root.

## Docker Registry Issue

The first Docker build failed while trying to pull images from:

```text
mcr.microsoft.com
```

Testing the registry endpoint with:

```bash
curl -i https://mcr.microsoft.com/v2/
```

returned an HTML ThreatLocker Web Control block page instead of container registry data.

Pulling another public image such as:

```bash
docker pull ubuntu:24.04
```

worked correctly.

This showed that Docker itself was functioning and that the issue was specific to access to Microsoft Container Registry.

After the blocking control was removed, Docker was able to pull the required .NET images.

Docker temporarily returned:

```text
Operation not permitted
```

after the security control was removed. Rebooting the VM restored normal Docker execution.

## Vulnerable Image Scan

Built the vulnerable image:

```bash
docker build -t devops-assessment:1.0.0 .
```

Scanned it with:

```bash
trivy image devops-assessment:1.0.0
```

Trivy reported both operating-system vulnerabilities and the application dependency vulnerability.

The application-level finding included:

```text
Newtonsoft.Json 9.0.1
CVE-2024-21907
Severity: HIGH
Fixed version: 13.0.1
```

## Remediation

Updated Newtonsoft.Json:

```bash
dotnet add package Newtonsoft.Json --version 13.0.1
```

Ran the dependency scan again:

```bash
dotnet list package --vulnerable --include-transitive
```

No vulnerable NuGet packages were reported.

Built the corrected image:

```bash
docker build -t devops-assessment:1.0.2 .
```

Verified:

- `/health` worked
- `/version` worked
- the container ran as `appuser`
- Docker reported the container as healthy

Verified the container user with:

```bash
docker exec devops-api whoami
```

Result:

```text
appuser
```

## Container Name Conflict

During local testing, an existing `devops-api` container caused a Docker name collision.

Resolved it with:

```bash
docker stop devops-api 2>/dev/null || true
docker rm devops-api 2>/dev/null || true
```

This same pattern was later used in the deployment workflow to make replacement of the container idempotent.

## Final Trivy Gate

Ran:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  --ignore-unfixed \
  --exit-code 1 \
  devops-assessment:1.0.2
```

Final result:

```text
HIGH: 0
CRITICAL: 0
```

## Why Run the Container as Non-Root?

Running the application as a non-root user reduces the impact of a container compromise.

If the application is exploited while running as root, the attacker has root privileges inside the container. Running as `appuser` limits those privileges and reduces the available attack surface.

To make this work, I created the user in the runtime stage and copied the published application files with ownership assigned to that user:

```dockerfile
COPY --from=build --chown=appuser:appuser /app/publish ./
USER appuser
```

This ensured the application files could be accessed by the non-root user.

---

# Part 3 — GitHub Actions

Created one workflow:

```text
.github/workflows/ci.yml
```

The workflow is triggered by:

- pushes to `main`
- pull requests targeting `main`
- manual `workflow_dispatch`

## Pipeline Order

The main pipeline follows:

```text
Format Check
→ Build and Test
→ Dependency Scan
→ Docker Build and Trivy Scan
→ Push to GHCR
→ Deploy
→ Verify
```

Separate jobs were used because they provide clearer visibility into which stage failed and allow explicit dependencies through `needs:`.

## Format Check

The first job restores the project and runs:

```bash
dotnet format DevOpsAssessment/DevOpsAssessment.csproj \
  --verify-no-changes \
  --no-restore
```

An `.editorconfig` file was added to define formatting rules.

Initially the format check detected CRLF line endings in `Program.cs`.

Running:

```bash
dotnet format DevOpsAssessment/DevOpsAssessment.csproj
```

corrected the formatting and allowed the check to pass.

## Build and Test

The next job depends on the format job using:

```yaml
needs: format
```

It performs:

```bash
dotnet restore
dotnet build --configuration Release --no-restore
dotnet test --configuration Release --no-build
```

## Dependency Scan

The dependency scan runs after build/test.

It uses:

```bash
dotnet list DevOpsAssessment/DevOpsAssessment.csproj \
  package \
  --vulnerable \
  --include-transitive
```

This is primarily used as a dependency vulnerability report.

## NuGet Caching

NuGet packages are cached with:

```yaml
uses: actions/cache@v4
```

The cache key is based on:

```text
hashFiles('**/*.csproj')
```

This avoids downloading unchanged NuGet packages repeatedly across workflow runs.

## Docker Build and Trivy Gate

The workflow builds the Docker image using the Git commit SHA as the image tag.

Example:

```text
devops-assessment:${{ github.sha }}
```

The image is then scanned using Trivy for:

```text
HIGH
CRITICAL
```

The Trivy step uses:

```text
exit-code: 1
```

which makes the workflow fail if a matching vulnerability is detected.

This is the main security gate before publishing the image.

## GHCR Publishing

After the security scan passes on a push to `main`, the workflow authenticates to GitHub Container Registry using:

```text
GITHUB_TOKEN
```

The image is tagged with:

- the Git commit SHA
- `latest`

and pushed to:

```text
ghcr.io/juanchareun/devops-assessment
```

The first GHCR push failed with:

```text
denied: installation not allowed to Create organization package
```

The workflow permissions were changed to allow package writes:

```yaml
permissions:
  contents: read
  packages: write
```

The Docker image was also given the OCI source label:

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/juanchareun/devops-assessment"
```

After these changes, the image published successfully.

## Manual Redeployment

The workflow includes:

```yaml
workflow_dispatch:
```

with a `redeploy_only` input.

When:

```text
redeploy_only = true
```

the build, dependency scan, and image build jobs are skipped.

The deploy job pulls the previously published:

```text
latest
```

image and redeploys it.

This was tested successfully from the GitHub Actions interface.

## `needs:` and Pipeline Gating

The workflow uses job dependencies similar to:

```text
format
  ↓
build
  ↓
dependency-scan
  ↓
docker-build-and-scan
  ↓
deploy
```

Using `needs:` ensures downstream stages do not proceed if an earlier required job fails.

---

# Workflow Diagram

A Mermaid pipeline diagram was added to:

```text
PIPELINE.md
```

The diagram shows:

- workflow triggers
- jobs
- `needs:` relationships
- security gates
- reporting stages
- GHCR publishing
- deployment
- manual redeployment

---

# Part 4 — Deploy Step

The deploy job runs after the Docker build and security gate succeeds on `main`.

The deployment process is:

```text
Pull image from GHCR
→ Stop old container
→ Remove old container
→ Start new container
→ Verify /health
→ Verify /version
```

## Container Replacement

Before starting the replacement container, the workflow runs:

```bash
docker stop devops-api 2>/dev/null || true
docker rm devops-api 2>/dev/null || true
```

This makes repeated deployments safe even if the previous container exists.

The new container is started with:

```bash
docker run -d \
  --name devops-api \
  -p 8080:8080 \
  -e APP_VERSION=<image-tag> \
  <GHCR-image>
```

## Deployment Failure — Incorrect Final Docker Stage

During the first deployment test, the container was created successfully but immediately exited.

The GitHub Actions log showed the container command as:

```text
bash
```

instead of:

```text
dotnet DevOpsAssessment.dll
```

Inspection of the Dockerfile showed that an additional:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
```

had accidentally been added after:

```dockerfile
ENTRYPOINT ["dotnet", "DevOpsAssessment.dll"]
```

Because Docker uses the final stage as the resulting image, the pushed image contained an empty ASP.NET runtime stage instead of the actual application.

The duplicate `FROM` was removed and the OCI source label was moved into the correct runtime stage.

After rebuilding, the container started correctly.

## Deployment Readiness Check

The first verification used a fixed:

```bash
sleep 5
```

before calling the application.

This was replaced with a readiness loop that repeatedly checks:

```text
/health
```

before checking the version endpoint.

If the service never becomes ready, the workflow prints:

```bash
docker ps -a
docker logs devops-api
```

and fails the deployment.

This provides better diagnostics than relying on a fixed delay.

## Version Verification

After the container becomes healthy, the workflow requests:

```text
/version
```

and confirms the returned version matches the image tag used for the deployment.

The final deployment workflow completed successfully.

The manual `workflow_dispatch` redeployment also completed successfully.

## Deployment Risk

The current deployment runs one container directly using `docker run`.

There is no automatic rollback if a newly deployed container becomes unhealthy after deployment, and there is currently no Docker restart policy.

A small improvement would be:

```bash
--restart unless-stopped
```

This would allow Docker to restart the service automatically after a process failure or host reboot.

A more complete solution would require health-aware deployment and rollback logic or a container orchestrator.

---

# Part 5 — Written Section

## 1. Would I Run Format, Test, and Security Scans in Parallel?


## 2. Where Do Secrets Live and What Is the Blast Radius?



## 3. What Is the Gap With `--ignore-unfixed`?



## 4. What Changes for Three Replicas Behind a Load Balancer?



# Final Result

The completed project includes:

- ASP.NET Core Web API
- `/health` endpoint
- `/version` endpoint
- multi-stage Dockerfile
- non-root container execution
- Docker health check
- dependency vulnerability scanning
- Trivy image security gate
- GitHub Actions CI/CD workflow
- NuGet caching
- GHCR publishing
- main-only deployment
- manual redeployment with `workflow_dispatch`
- deployment health and version verification
- Mermaid pipeline diagram

The final CI/CD pipeline and manual redeployment both completed successfully.
