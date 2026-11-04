# Lab 6 – Build automation and Docker image builds

Report for Applied DevOps Lab 6, Tasks 0–7. Task 8 (containerizing the project) is out of scope for this document.

- Demo repository: <https://github.com/ceciliagalvaoo/docker-build-demo>
- Author: Cecília Galvão (GitHub: `ceciliagalvaoo`)

All links in this report are absolute so that they keep working when the file is copied to another repository. Every command output shown here was captured from a real run; anything that was skipped or failed is stated explicitly.

---

## Task 0 – Preparation

### Repository

A new **public** repository `docker-build-demo` was created on GitHub and cloned to `C:\Users\Inteli\Documents\GitHub\docker-build-demo` (accessed from WSL as `/mnt/c/Users/Inteli/Documents/GitHub/docker-build-demo`). It is completely separate from the project repository.

```text
$ git remote -v
origin  https://github.com/ceciliagalvaoo/docker-build-demo.git (fetch)
origin  https://github.com/ceciliagalvaoo/docker-build-demo.git (push)

$ gh repo view ceciliagalvaoo/docker-build-demo --json name,visibility,isEmpty
{"isEmpty":true,"name":"docker-build-demo","visibility":"PUBLIC"}      # before the first commit

$ git log --oneline -1
0a94fbe chore: initial commit                                          # empty initial commit on main
```

Workflow (same as Lab 5): one branch per task (`task-0-preparation`, `task-1-...`, …), a pull request into `main`, then a merge.

### Environment

| Component | Where | Version / result |
|-----------|-------|------------------|
| OS | WSL2 | Ubuntu 24.04.3 LTS (kernel 6.6.87.2-microsoft-standard-WSL2) on Windows |
| Docker | WSL2 (Docker Engine, **no Docker Desktop**) | Client 28.5.1, Engine 28.5.1, API 1.51, containerd v1.7.28, runc 1.3.0 |
| BuildKit | WSL2 | buildx v0.29.1; builder `default` (driver `docker`) running BuildKit v0.25.1, platforms `linux/amd64 (+3), linux/386` |
| .NET SDK | WSL2 | 10.0.403 (installed during Task 0, see below) |
| JDK | WSL2 | OpenJDK 21.0.12.1 (`javac 21.0.12.1`) |
| Maven | – | not installed globally (not needed; the project uses the Maven wrapper `mvnw`) |
| GitHub CLI | WSL2 | gh 2.102.0, logged in as `ceciliagalvaoo` |
| Git | Windows | 2.52.0.windows.1, `core.autocrlf=true` (system config) |

Docker check:

```text
$ docker version            # Client 28.5.1 / Server 28.5.1 (see table)
$ docker run --rm hello-world
Hello from Docker!
...
$ docker buildx ls
NAME/NODE       DRIVER/ENDPOINT   STATUS    BUILDKIT   PLATFORMS
default*        docker
 \_ default      \_ default       running   v0.25.1    linux/amd64 (+3), linux/386
desktop-linux                     error
```

The `desktop-linux` builder/context is a leftover of a former Docker Desktop installation and is not used. All Docker commands in this lab run against the Docker Engine inside WSL2 (`unix:///var/run/docker.sock`).

### Missing SDK and how it was resolved

At the start, the **.NET SDK 10 was missing**: WSL had no `dotnet`, and Windows only had SDK 9.0.306. The lab allows three alternatives (install locally, use a Codespace, or skip the native builds and rely on the Docker builds that contain their own SDK). The SDK was installed locally in WSL, so that the native builds (Task 1) and the Docker builds run on the same Linux machine.

`apt` required a sudo password, so the official Microsoft install script was used instead (user-level install, no root):

```text
$ curl -fsSL https://dot.net/v1/dotnet-install.sh -o dotnet-install.sh
$ bash dotnet-install.sh --channel 10.0 --install-dir "$HOME/.dotnet"
dotnet-install: Installed version is 10.0.403
dotnet-install: Installation finished successfully.

$ export DOTNET_ROOT=$HOME/.dotnet PATH=$HOME/.dotnet:$PATH
$ dotnet --list-sdks
10.0.403 [/home/inteli/.dotnet/sdk]
```

### Notes for the next tasks

- **Line endings:** Windows Git uses `core.autocrlf=true`. A shell script such as `mvnw` checked out with CRLF fails on Linux (`/bin/sh^M: bad interpreter`). A `.gitattributes` file that forces LF (and CRLF only for `*.cmd`) is added in Task 1.
- Native builds and Docker builds are executed in **WSL2**. They were not repeated on native Windows.

---

## Task 1 – Sample applications and native builds

All commands in this task were run **natively in WSL2 (Ubuntu 24.04)** with .NET SDK 10.0.403 and OpenJDK 21.0.12, on the branch `task-1-native-builds`.

### 1.1 .NET

#### Generation

```text
$ mkdir dotnet-app && cd dotnet-app
$ dotnet new webapi  -n DemoApi       -o src/DemoApi          # The template "ASP.NET Core Web API" was created successfully.
$ dotnet new xunit   -n DemoApi.Tests -o tests/DemoApi.Tests  # The template "xUnit Test Project" was created successfully.
$ dotnet add tests/DemoApi.Tests reference src/DemoApi/DemoApi.csproj
Reference `..\..\src\DemoApi\DemoApi.csproj` added to the project.
$ dotnet new sln -n DemoApp                                   # creates DemoApp.slnx on .NET 10
$ dotnet sln add src/DemoApi tests/DemoApi.Tests
Project `src/DemoApi/DemoApi.csproj` added to the solution.
Project `tests/DemoApi.Tests/DemoApi.Tests.csproj` added to the solution.
$ dotnet new gitignore
$ dotnet new globaljson --sdk-version 10.0.100 --roll-forward latestFeature
```

`Directory.Build.props` was added exactly as given in the lab (`Version=0.1.0`, `TreatWarningsAsErrors=true`). The generated `.gitignore` already contains `[Bb]in/`, `[Oo]bj/` and `publish/`, so nothing had to be added. `git status --ignored` confirms that `bin/`, `obj/` and `publish/` are ignored.

`global.json` pins the SDK to **major version 10**: `version 10.0.100` with `rollForward: latestFeature` accepts any 10.0.x SDK (feature band 100 or newer). This is deliberate. A strict pin to the local `10.0.403` would break builds on machines and Docker images that have another 10.0 feature band, for example the SDK inside `mcr.microsoft.com/dotnet/sdk:10.0`.

#### Native build (the four commands)

| Step | Command | Result | Wall time |
|------|---------|--------|-----------|
| restore | `dotnet restore` | both projects restored, exit 0 | 2.75 s |
| compile | `dotnet build --configuration Release --no-restore` | `Build succeeded. 0 Warning(s) 0 Error(s)`, exit 0 | 13.05 s |
| test | `dotnet test --configuration Release --no-build` | `Passed! - Failed: 0, Passed: 1, Skipped: 0, Total: 1`, exit 0 | 2.85 s |
| publish | `dotnet publish src/DemoApi -c Release -o publish --no-build` | `DemoApi -> .../dotnet-app/publish/`, exit 0 | 2.30 s |

The timings come from `/usr/bin/time` with a warm NuGet cache, because the packages had already been restored by `dotnet new`. The `--no-restore` and `--no-build` flags make every command reuse the output of the previous one instead of repeating it. The Dockerfiles use the same split.

#### Running the published app

```text
$ dotnet publish/DemoApi.dll
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5000
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production

$ curl http://localhost:5000/weatherforecast
[{"date":"2026-11-05","temperatureC":45,"summary":"Bracing","temperatureF":112},{"date":"2026-11-06","temperatureC":31,"summary":"Hot","temperatureF":87}, ... ]
HTTP 200
```

The `publish/` folder is **916 KB** (`du -sh`; 927,327 bytes). It contains only what is needed to run the app on an installed ASP.NET Core runtime: `DemoApi.dll` (28,672 B), `DemoApi.pdb`, the `DemoApi` native launcher, `DemoApi.deps.json`, `DemoApi.runtimeconfig.json`, `appsettings*.json`, `web.config`, `DemoApi.staticwebassets.endpoints.json` and the two NuGet dependencies `Microsoft.AspNetCore.OpenApi.dll` and `Microsoft.OpenApi.dll`. It contains **no** source code, no `obj/` and no SDK. This is exactly what the final image in Task 5 copies.

#### Debug vs Release

Both configurations were built (`dotnet build -c Debug --no-restore` / `-c Release`), and the effective MSBuild properties were read with `dotnet msbuild -getProperty:...`:

| | Debug | Release |
|---|---|---|
| `Optimize` | `false` | `true` |
| `DefineConstants` | `TRACE;DEBUG` | `TRACE;RELEASE` |
| `DebugType` | `portable` | `portable` |
| `DemoApi.dll` | 29,696 B | 28,672 B |
| `DemoApi.pdb` | 34,172 B | 32,468 B |
| `bin/<cfg>/net10.0/` | 924 KB | 916 KB |

**Note:** Debug compiles without IL optimizations and defines the `DEBUG` symbol, so `#if DEBUG` code and `Debug.Assert` are included. The resulting IL is easier to step through in a debugger. Release turns on the compiler optimizations and leaves out the `DEBUG` code, which makes the DLL slightly smaller (1 KB here, since the app is tiny) and faster. Both produce a portable `.pdb` symbol file by default; it is not needed at runtime. Images and deployments always use Release.

#### .NET configuration files in this project

| File | What it does here |
|------|-------------------|
| `src/DemoApi/DemoApi.csproj` | `Sdk="Microsoft.NET.Sdk.Web"` (web project), `TargetFramework net10.0`, `Nullable`/`ImplicitUsings` enabled, one package: `Microsoft.AspNetCore.OpenApi 10.0.14` |
| `tests/DemoApi.Tests/DemoApi.Tests.csproj` | Test project (`IsPackable=false`) with xunit 2.9.3, xunit.runner.visualstudio 3.1.4, Microsoft.NET.Test.Sdk 17.14.1, coverlet.collector 6.0.4 and a `ProjectReference` to `DemoApi.csproj` |
| `DemoApp.slnx` | New XML solution format of .NET 10. It groups the two projects in the folders `/src/` and `/tests/`, so `dotnet restore/build/test` at the folder root handles both |
| `global.json` | Pins the SDK major version (10.0.100+, `latestFeature`) |
| `Directory.Build.props` | Imported automatically by MSBuild into every project below `dotnet-app/`. It sets `Version=0.1.0` (assembly/file version) and `TreatWarningsAsErrors=true` (a compiler warning breaks the build) for both projects |
| `nuget.config` | Not created; the default source `nuget.org` is used |
| `src/DemoApi/appsettings*.json`, `Properties/launchSettings.json` | Runtime configuration and the local `dotnet run` profiles. `launchSettings.json` is not used by `dotnet DemoApi.dll` or in containers |

### 1.2 Java

#### Generation

```text
$ curl https://start.spring.io/starter.zip \
    -d type=maven-project -d language=java -d javaVersion=21 \
    -d groupId=hr.demo -d artifactId=demo -d name=demo -d dependencies=web \
    -o demo.zip
```

`unzip` is not installed in this WSL distribution, so the archive was extracted with `python3 -m zipfile -e demo.zip java-app`. This tool does not keep Unix file modes, so the executable bit of `mvnw` was restored with `chmod +x mvnw`.

The generated project contains `pom.xml`, the Maven wrapper (`mvnw`, `mvnw.cmd`, `.mvn/wrapper/maven-wrapper.properties`: wrapper 3.3.4, `distributionType=only-script`, Maven **3.10.0**), `DemoApplication.java`, one test `DemoApplicationTests.java` (a `contextLoads` test), `application.properties`, a `.gitignore` (ignores `target/` and `HELP.md`) and a `.gitattributes` (`/mvnw text eol=lf`, `*.cmd text eol=crlf`). The `.gitattributes` solves the CRLF problem noted in Task 0.

Spring Boot 4 renamed the `web` starter: the generated `pom.xml` uses `spring-boot-starter-webmvc` and `spring-boot-starter-webmvc-test`.

#### Native build

Maven is **not** installed globally (`mvn: command not found`). The wrapper runs Maven 3.10.0 from `~/.m2/wrapper/dists/apache-maven-3.10.0/`. That distribution, and the dependencies in `~/.m2/repository`, were already cached on this machine from earlier work (dated 2026-10-10), so no downloads happened in this run. The 26 s below is therefore a **warm-cache** time. On a fresh machine, the wrapper first downloads Maven and then all the dependencies.

```text
$ ./mvnw -B -ntp clean verify
[INFO] --- clean:3.5.0:clean (default-clean) @ demo ---
[INFO] --- resources:3.5.0:resources (default-resources) @ demo ---
[INFO] --- compiler:3.15.0:compile (default-compile) @ demo ---
[INFO] --- resources:3.5.0:testResources (default-testResources) @ demo ---
[INFO] --- compiler:3.15.0:testCompile (default-testCompile) @ demo ---
[INFO] --- surefire:3.5.6:test (default-test) @ demo ---
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] --- jar:3.5.1:jar (default-jar) @ demo ---
[INFO] --- spring-boot:4.1.2:repackage (repackage) @ demo ---
[INFO] Building jar: .../java-app/target/demo-0.0.1-SNAPSHOT.jar
[INFO] BUILD SUCCESS
[INFO] Total time:  26.179 s
exit=0
```

The plugin executions show the lifecycle: `clean` (separate lifecycle) → `resources` + `compile` (compile phase) → `testResources` + `testCompile` + `surefire:test` (test phase) → `jar:jar` + `spring-boot:repackage` (package phase) → `verify` (no extra plugin bound in this project).

```text
$ ls target/
classes/  demo-0.0.1-SNAPSHOT.jar  demo-0.0.1-SNAPSHOT.jar.original  generated-sources/
generated-test-sources/  maven-archiver/  maven-status/  surefire-reports/  test-classes/
```

- `demo-0.0.1-SNAPSHOT.jar`: **20 MB** (19,963,208 B), the executable "fat" JAR made by `spring-boot:repackage`, with all dependencies (Tomcat, Spring) under `BOOT-INF/lib/`
- `demo-0.0.1-SNAPSHOT.jar.original`: 2,578 B, the plain JAR made by `jar:jar` (only our classes)
- `surefire-reports/`: `TEST-hr.demo.demo.DemoApplicationTests.xml` and `.txt` (test reports)

#### Running the JAR

```text
$ java -jar target/demo-0.0.1-SNAPSHOT.jar
 :: Spring Boot ::                (v4.1.2)
... Tomcat started on port 8080 (http) with context path '/'
... Started DemoApplication in 4.78 seconds (process running for 6.032)

$ curl http://localhost:8080/
{"timestamp":"2026-11-04T15:57:30.831Z","status":404,"error":"Not Found","path":"/"}
HTTP 404
```

The template has **no controller**, and the lab writes no application logic. The JSON 404 is produced by Spring Boot's own error handling and proves that the embedded Tomcat is up and that the Spring application answers HTTP requests. The same check is used for the Java containers.

#### `pom.xml` sections

| Section | Content in this project | What it does |
|---------|------------------------|--------------|
| `<parent>` | `org.springframework.boot:spring-boot-starter-parent:4.1.2` | Inherits the dependency versions (a BOM in `<dependencyManagement>`), plugin versions and plugin configuration from Spring Boot. That is why our dependencies and plugins have no `<version>` |
| `<groupId>/<artifactId>/<version>` | `hr.demo:demo:0.0.1-SNAPSHOT` | The coordinates of the artifact. They produce the name `demo-0.0.1-SNAPSHOT.jar` |
| `<properties>` | `java.version=21` | Read by the parent, which sets the compiler `release` to 21: bytecode for Java 21 |
| `<dependencies>` | `spring-boot-starter-webmvc` (compile scope); `spring-boot-starter-webmvc-test` (`test` scope) | What the app needs: Spring MVC + embedded Tomcat at runtime; JUnit, Spring Test and MockMvc only for compiling and running tests (not in the final JAR) |
| `<build><plugins>` | `spring-boot-maven-plugin` | Adds the `repackage` goal to the `package` phase, which turns the plain JAR into an executable fat JAR with a `Main-Class` launcher |

### Build steps per ecosystem (reused in the Dockerfiles)

| Stage | .NET | Java (Maven) |
|-------|------|--------------|
| restore / resolve dependencies | `dotnet restore` | `./mvnw -B -ntp dependency:go-offline` (implicit in every phase) |
| compile | `dotnet build -c Release --no-restore` | `./mvnw -B -ntp compile` |
| test | `dotnet test -c Release --no-build` | `./mvnw -B -ntp test` |
| package / publish | `dotnet publish src/DemoApi -c Release -o publish --no-build` | `./mvnw -B -ntp package` (or `-DskipTests`) |
| all in one | the four commands above | `./mvnw -B -ntp clean verify` |
| artifact | `publish/` folder (916 KB) | `target/demo-0.0.1-SNAPSHOT.jar` (20 MB) |

### Versioning of generated files

The build outputs are excluded from Git: `dotnet-app/.gitignore` (`[Bb]in/`, `[Oo]bj/`, `publish/`) and `java-app/.gitignore` (`target/`, `HELP.md`). The repository uses `core.filemode=false` (NTFS), so the executable bit of `mvnw` is stored explicitly in the Git index with `git update-index --chmod=+x java-app/mvnw`.

## Task 2 – Anatomy of a Dockerfile and of `docker build`

_Pending._

## Task 3 – Naive Dockerfiles

_Pending._

## Task 4 – Layer cache and `.dockerignore`

_Pending._

## Task 5 – Multi-stage builds

_Pending._

## Task 6 – ARG, LABEL, test stage and BuildKit features

_Pending._

## Task 7 – Tagging, publishing and running the image

_Pending._

---

## Final checklist (Tasks 0–7)

| # | Item | Status |
|---|------|--------|
| 0 | Public `docker-build-demo` repository created and cloned; tools checked | ✅ Done |
| 1 | .NET and Java apps built with their native tools; build steps listed | ✅ Done (native, WSL2) |
| 2 | Task 2 questions answered | ⏳ Pending |
| 3 | Naive Dockerfiles built and analyzed with `docker history` | ⏳ Pending |
| 4 | Cache behavior observed and the table filled in | ⏳ Pending |
| 4 | `.dockerignore` files created | ⏳ Pending |
| 5 | Multi-stage Dockerfiles for .NET and Java, running as non-root | ⏳ Pending |
| 5 | Image comparison table filled in with own measurements | ⏳ Pending |
| 6 | `ARG`, `LABEL`, a `test` stage and a cache mount used | ⏳ Pending |
| 6 | Secret leak demonstrated, and a secret mount used | ⏳ Pending |
| 6 | Multi-platform build | ⏳ Optional |
| 7 | Image tagged, pushed to `ghcr.io`, pulled and run, also by digest | ⏳ Pending |
| 8 | Project containerized and documented | ➖ Out of scope for this report (separate deliverable in the project repository) |
