# ----- Build stage -----
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

WORKDIR /src

COPY DevOpsAssessment.csproj ./
RUN dotnet restore

COPY . ./

RUN dotnet publish \
    -c Release \
    -o /app/publish \
    --no-restore

# ----- Runtime stage -----
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime

WORKDIR /app

RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/* \
    && adduser --disabled-password --gecos "" appuser

COPY --from=build --chown=appuser:appuser /app/publish ./

USER appuser

ENV ASPNETCORE_URLS=http://+:8080

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s \
    CMD curl -f http://localhost:8080/health || exit 1

ENTRYPOINT ["dotnet", "DevOpsAssessment.dll"]

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime

WORKDIR /app

LABEL org.opencontainers.image.source="https://github.com/juanchareun/devops-assessment"
