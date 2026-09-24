# CI/CD Pipeline

```mermaid
flowchart LR
    A[Push / PR to main] --> B[Format Check]
    B --> C[Build and Test]
    C --> D[Dependency Scan]
    D --> E[Docker Build]
    E --> F{Trivy Scan}

    F -->|Fail| G[Stop]
    F -->|Pass| H[Push to GHCR]
    H --> I[Deploy]
    I --> J[Check /health]
    J --> K[Check /version]

    L[Manual workflow_dispatch] --> M[Pull latest image]
    M --> I
```

The jobs are chained with `needs:` in `ci.yml`.

The dependency scan is mainly for reporting. The Trivy scan is the security gate before the image is pushed and deployed.

The manual workflow can redeploy the last published image without rebuilding it.
