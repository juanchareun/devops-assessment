# DevOps Assessment Worklog

## Environment

- Ubuntu Linux VM
- .NET 8 SDK
- Docker
- Trivy
- Git/GitHub over SSH

## Part 1 - ASP.NET Core Service

Created an ASP.NET Core Web API with two endpoints:

- `GET /health` returns HTTP 200 with a small JSON health response.
- `GET /version` reads the `APP_VERSION` environment variable and defaults to `dev`.

Verified both endpoints locally using curl.

Verified `/version` both without `APP_VERSION` and with `APP_VERSION=1.0.0`.

## Part 2 - Vulnerability Scanning

### Dependency baseline

Pending.

### Notes / Issues Encountered

During the initial Docker build, Docker successfully located the Dockerfile but failed while retrieving the .NET 8 SDK image from `mcr.microsoft.com`. The registry request returned an unexpected `text/html` content type. This appears to be a registry/network-related issue rather than a Dockerfile parsing issue and is being investigated.
