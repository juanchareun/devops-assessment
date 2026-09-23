# DevOps Assessment Worklog

## Environment

- Ubuntu Linux VM
- .NET 8 SDK
- Docker
- Trivy
- Git/GitHub over SSH

## Part 1 - ASP.NET Core Service

Created an ASP.NET Core Web API with two endpoints:

- `GET /health` returns HTTP 200 with a small JSON response.
- `GET /version` reads the `APP_VERSION` environment variable and defaults to `dev`.

Verified both endpoints locally using `curl`.

Verified `/version` with and without `APP_VERSION` set.

```bash
APP_VERSION=1.0.0 dotnet run
```

---

## Part 2 - Vulnerability Scanning

### Dependency Baseline

Ran:

```bash
dotnet list package --vulnerable --include-transitive
```

The initial dependency scan returned no known vulnerable packages.

### Intentionally Vulnerable Package

Added:

```bash
dotnet add package Newtonsoft.Json --version 9.0.1
```

The scan detected a HIGH severity vulnerability.

### Docker

Created a multi-stage Dockerfile using:

- .NET 8 SDK for the build stage
- ASP.NET 8 runtime for the final stage
- Release build
- Non-root `appuser`
- Port `8080`
- Docker `HEALTHCHECK`

### Issues / Troubleshooting

The first Docker build failed while pulling from `mcr.microsoft.com`.

Testing showed ThreatLocker Web Control was returning an HTML block page instead of container registry data.

After removing the blocking control, Docker was able to pull the .NET images successfully.

Docker also temporarily returned:

```text
Operation not permitted
```

after the agent was removed.

The VM was rebooted and the environment recovered.

### Vulnerable Image Scan

Built the vulnerable image:

```bash
docker build -t devops-assessment:1.0.0 .
```

Scanned with:

```bash
trivy image devops-assessment:1.0.0
```

Trivy detected:

- OS-level vulnerabilities
- `Newtonsoft.Json 9.0.1`
- HIGH severity
- `CVE-2024-21907`
- Fixed version `13.0.1`

### Remediation

Updated the package:

```bash
dotnet add package Newtonsoft.Json --version 13.0.1
```

The dependency scan returned no vulnerable packages.

Built the hardened image:

```bash
docker build -t devops-assessment:1.0.2 .
```

Verified:

- `/health` works
- `/version` works
- Container runs as `appuser`
- Docker health status becomes `healthy`

### Container Name Conflict

An old `devops-api` container caused a name conflict.

Resolved with:

```bash
docker stop devops-api 2>/dev/null || true
docker rm devops-api 2>/dev/null || true
```

### Final Security Gate

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
0 HIGH
0 CRITICAL
```
