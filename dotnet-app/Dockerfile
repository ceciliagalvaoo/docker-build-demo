# syntax=docker/dockerfile:1

# Global ARG: declared before the first FROM, so it can only be used in FROM lines.
ARG DOTNET_VERSION=10.0

############ Stage 1: build ############
FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION} AS build
WORKDIR /src

COPY Directory.Build.props ./
COPY src/DemoApi/DemoApi.csproj src/DemoApi/
# Cache mount: the NuGet download cache lives outside the image layers.
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet restore src/DemoApi/DemoApi.csproj

COPY src/DemoApi/ src/DemoApi/
# Declared here (not right after FROM) so that a new version does not invalidate the restore above.
ARG APP_VERSION=0.0.0
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet publish src/DemoApi/DemoApi.csproj \
        -c Release -o /app/publish --no-restore -p:Version=${APP_VERSION}

############ Test stage (only runs with --target test) ############
FROM build AS test
COPY tests/DemoApi.Tests/ tests/DemoApi.Tests/
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet test tests/DemoApi.Tests/DemoApi.Tests.csproj -c Release

############ Stage 2: runtime ############
FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION} AS final
# ARGs are scoped to a stage: APP_VERSION must be declared again to be used here.
ARG APP_VERSION=0.0.0
LABEL org.opencontainers.image.version=${APP_VERSION} \
      org.opencontainers.image.source="https://github.com/ceciliagalvaoo/docker-build-demo" \
      org.opencontainers.image.description="Demo API" \
      org.opencontainers.image.licenses="MIT"
WORKDIR /app

COPY --from=build /app/publish .

EXPOSE 8080
USER $APP_UID
ENTRYPOINT ["dotnet", "DemoApi.dll"]
