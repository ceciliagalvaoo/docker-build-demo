# syntax=docker/dockerfile:1

############ Stage 1: build ############
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

COPY Directory.Build.props ./
COPY src/DemoApi/DemoApi.csproj src/DemoApi/
RUN dotnet restore src/DemoApi/DemoApi.csproj

COPY src/DemoApi/ src/DemoApi/
RUN dotnet publish src/DemoApi/DemoApi.csproj \
        -c Release -o /app/publish --no-restore

############ Stage 2: runtime ############
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS final
WORKDIR /app

COPY --from=build /app/publish .

EXPOSE 8080
USER $APP_UID
ENTRYPOINT ["dotnet", "DemoApi.dll"]
