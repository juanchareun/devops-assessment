# DevOps Assessment Worklog

## Environment

I used an Ubuntu VM for the assessment.

Tools:

- .NET 8 SDK
- Docker
- Trivy
- Git/GitHub
- GitHub Container Registry

---

# Part 1 — Scaffold a real service

Created an ASP.NET Core Web API with:

- `GET /health`
- `GET /version`

`/health` returns a small JSON response and HTTP 200.

`/version` reads `APP_VERSION` and defaults to `dev`.

Tested locally with:

```bash
dotnet run
curl http://localhost:5205/health
curl http://localhost:5205/version
```

I also tested the environment variable override:

```bash
APP_VERSION=1.0.0 dotnet run
```

`/version` returned `1.0.0`, so the override was working.

---

# Part 2 — Vulnerability scanning

## Dependency scan

Started with:

```bash
dotnet list package --vulnerable --include-transitive
```

Then added the intentionally vulnerable package:

```bash
dotnet add package Newtonsoft.Json --version 9.0.1
```

Running the scan again showed a HIGH severity vulnerability in Newtonsoft.Json.

The finding was:

```text
Newtonsoft.Json 9.0.1
CVE-2024-21907
Fixed in 13.0.1
```

## Docker image

Created a multi-stage Dockerfile with:

- .NET 8 SDK build stage
- ASP.NET 8 runtime stage
- Release publish
- non-root `appuser`
- port 8080
- health check against `/health`

### Issue: Microsoft Container Registry was blocked

The first Docker build failed while pulling from:

```text
mcr.microsoft.com
```

I tested the registry directly:

```bash
curl -i https://mcr.microsoft.com/v2/
```

Instead of registry data, the response was a ThreatLocker Web Control block page.

I also tested Docker Hub:

```bash
docker pull ubuntu:24.04
```

That worked, which narrowed the problem down to access to MCR rather than Docker itself.

After removing the blocking control, the .NET images could be pulled normally.

Docker briefly started returning:

```text
Operation not permitted
```

after the security control was removed. Rebooting the VM cleared that issue.

## Trivy scan

Built the vulnerable image:

```bash
docker build -t devops-assessment:1.0.0 .
```

Then scanned it:

```bash
trivy image devops-assessment:1.0.0
```

Trivy showed OS findings as well as the Newtonsoft.Json vulnerability.

I updated the package:

```bash
dotnet add package Newtonsoft.Json --version 13.0.1
```

The NuGet vulnerability scan was clean after the update.

I rebuilt the image and verified:

```bash
curl http://localhost:8080/health
curl http://localhost:8080/version
docker exec devops-api whoami
```

`whoami` returned:

```text
appuser
```

## Why do we care if the container runs as root, and what did you have to change to make `USER appuser` actually work?

Running the application as root gives it more privileges than it needs.

If the application was compromised, running as a non-root user would limit what that process could do inside the container.

To make `appuser` work, I created the user in the runtime stage and copied the published files with the correct ownership:

```dockerfile
COPY --from=build --chown=appuser:appuser /app/publish ./
USER appuser
```

That let the application run normally without root.

## Container name conflict

While testing locally, I hit a container name collision because `devops-api` already existed.

I fixed it with:

```bash
docker stop devops-api 2>/dev/null || true
docker rm devops-api 2>/dev/null || true
```

I reused the same pattern later in the deploy job.

## Final gate

Ran:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  --ignore-unfixed \
  --exit-code 1 \
  devops-assessment:1.0.2
```

Final filtered result:

```text
HIGH: 0
CRITICAL: 0
```

---

# Part 3 — GitHub Actions

Created one workflow:

```text
.github/workflows/ci.yml
```

It runs on:

- push to `main`
- pull request to `main`
- manual `workflow_dispatch`

I split the workflow into separate jobs so each stage has its own status and I can control the order with `needs:`.

The main flow is:

```text
Format
→ Build/Test
→ Dependency Scan
→ Docker Build + Trivy
→ Push to GHCR
→ Deploy
```

## Format

The first job runs:

```bash
dotnet format DevOpsAssessment/DevOpsAssessment.csproj --verify-no-changes --no-restore
```

I added an `.editorconfig` to keep formatting consistent.

The first format run found line-ending differences in `Program.cs`.

I ran `dotnet format`, committed the result, and the check passed afterward.

## Build/Test

The build job depends on the format job.

It restores, builds in Release, then runs:

```bash
dotnet test
```

## Dependency scan

The next job runs:

```bash
dotnet list DevOpsAssessment/DevOpsAssessment.csproj package --vulnerable --include-transitive
```

I treat this mainly as a report step.

## Caching

I added NuGet caching using:

```yaml
actions/cache@v4
```

with the project files included in the cache key.

## Docker build and Trivy gate

The Docker image is built using the Git commit SHA as the tag.

Trivy scans the image for:

```text
HIGH
CRITICAL
```

and uses:

```text
exit-code: 1
```

so the workflow stops if the gate fails.

## GHCR

Images are only pushed on `main`.

The workflow tags the image with:

- the commit SHA
- `latest`

### Issue: first GHCR push failed

The initial push failed with:

```text
denied: installation not allowed to Create organization package
```

I changed the workflow permissions to:

```yaml
permissions:
  contents: read
  packages: write
```

I also added the repository source label to the Dockerfile.

After that, the image pushed successfully.

## Manual redeploy

I added a `workflow_dispatch` input called:

```text
redeploy_only
```

When that is set to `true`, the CI/build jobs are skipped and the deploy job pulls the last published `latest` image.

I tested this manually from the Actions tab and it completed successfully.

## Why `needs:` matters

The jobs are chained with `needs:` so a failed required stage prevents the later jobs from running.

That is especially important before image publishing and deployment.

---

# Workflow diagram

The pipeline diagram is in:

```text
PIPELINE.md
```

It shows the triggers, job dependencies, Trivy gate, GHCR publishing, deployment path, and manual redeploy path.

---

# Part 4 — Deploy step

The deploy job runs on the GitHub-hosted Actions runner.

It:

1. Logs in to GHCR
2. Pulls the image
3. Stops/removes the existing container
4. Starts the new container
5. Checks `/health`
6. Checks `/version`

The container is started with `APP_VERSION` set to the deployed image tag.

## Issue: container immediately exited

The first deployment created the container, but it exited immediately.

The Actions output showed the command as:

```text
bash
```

instead of the application entrypoint.

I checked the Dockerfile and found that I had accidentally added another:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
```

after the real runtime stage.

Because Docker uses the last stage as the final image, I was publishing an empty runtime image instead of the application image.

I removed the duplicate stage and rebuilt.

The next deployment started correctly.

## Readiness check

My first version just waited five seconds before calling `/health`.

That worked poorly because it assumed the application would always start within that exact time.

I changed it to retry `/health` for up to 60 seconds.

If the application never becomes ready, the workflow prints:

```bash
docker ps -a
docker logs devops-api
```

before failing.

## Deployment risk

### What's the actual risk of `docker run` with no restart policy and no health-based rollback, and what's the smallest change that reduces it?

The main risk is that if the container crashes or the host restarts, the application may stay down.

There is also no automatic rollback if a bad image starts successfully but later becomes unhealthy.

The smallest improvement I would make is:

```bash
--restart unless-stopped
```

That would at least restart the container after a crash or host reboot.

It would not solve health-based rollback, but it is a simple improvement over the current setup.

---

# Part 5 — Written section

## Format check, test, security scan — if all three could run in parallel instead of sequentially, would you? What do you gain, what do you risk?

I probably wouldn't run everything in parallel by default.

The main benefit would be speed. If format, tests, and security checks all started at the same time, I would get feedback faster.

The downside is that I could end up running jobs that were never needed. For example, if the formatting check fails immediately, there is not much value in continuing to spend time and runner resources on later stages.

For this project I preferred keeping the flow sequential because it makes it very clear where the pipeline failed and why the next stage did not run.

---

## Where do secrets live in your pipeline? What's the blast radius if one leaks into a log line?

I do not keep secrets directly in the repository or inside the workflow file.

For GHCR, I am using GitHub's `GITHUB_TOKEN`, which GitHub provides to the workflow automatically.

If I needed another credential, I would store it in GitHub Actions Secrets rather than putting it in the YAML.

If a secret did get exposed in a log, how bad it is would depend on what that secret is allowed to do.

For example, if someone got a token that could write to GHCR, they could potentially push or replace container images.

---

## Your Trivy gate uses `--ignore-unfixed`. A base-image CVE that had no fix gets one next month. Nothing in your pipeline catches that automatically. What's the gap, how do you close it?

The gap is that my pipeline mainly runs when I make a change.

If a vulnerability has no fix today, Trivy ignores it because of `--ignore-unfixed`.

If a fix becomes available next month but I have not changed the project, nothing automatically causes that image to be scanned again.

The simplest way I would improve that is by adding a scheduled workflow that runs Trivy regularly, even when there has not been a new commit.

I would also rebuild the image periodically so it picks up updates from the base image.

That way I am not depending only on code changes to find newly fixable vulnerabilities.

---

## If this needed 3 replicas behind a load balancer instead of one `docker run`, what's the smallest realistic next step — and what would you explicitly not try to solve with a Dockerfile change alone?

At that point I would move it to something that can actually manage multiple containers.

The next realistic step would be something like Kubernetes.

I would want the platform to handle things like keeping three instances running, checking whether they are healthy, replacing failed containers, and sending traffic between them.

I would not try to solve that inside the Dockerfile.

The Dockerfile should describe how to build and run one copy of the application.

Things like replicas, load balancing, failover, and rolling deployments belong in the deployment platform instead.

---

## Security gate validation

I tested the Trivy gate on a separate branch by adding `Newtonsoft.Json 9.0.1` again and opening a pull request.

The Docker build completed, but the Trivy scan found `CVE-2024-21907` as HIGH severity and exited with code 1.

Because the scan failed, the later GHCR publishing steps did not run.

This confirmed that the security gate actually stops the pipeline when a HIGH severity vulnerability is present.

# Result

The final pipeline successfully:

- formats and builds the project
- runs the dependency report
- builds the container
- blocks on HIGH/CRITICAL Trivy findings
- pushes images to GHCR
- deploys on `main`
- verifies `/health` and `/version`
- supports manual redeployment without another build
