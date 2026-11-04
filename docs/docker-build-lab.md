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

Task 2 is mostly theory. To answer its questions with real observations instead of only the lab's tables, a small **throwaway** Dockerfile was built against the real `dotnet-app/` context. It uses one instruction of each kind and is **not committed**: it only lives in this report. Base image `alpine:3.20`, Docker Engine 28.5.1 / BuildKit v0.25.1 on WSL2.

```dockerfile
# Dockerfile.anatomy (experiment only)
FROM alpine:3.20
WORKDIR /app
COPY Directory.Build.props global.json ./
RUN wc -c Directory.Build.props global.json > sizes.txt
ARG BUILD_NOTE=demo
ENV APP_ENV=lab
EXPOSE 8080
USER nobody
ENTRYPOINT ["cat"]
CMD ["sizes.txt"]
```

### 2.2 / 2.3 Reading a real build (`--progress=plain`)

First build (`docker build --progress=plain -f Dockerfile.anatomy -t anatomy-demo .` inside `dotnet-app/`):

```text
#1 [internal] load build definition from Dockerfile.anatomy        ← the Dockerfile is read and parsed (step 2)
#1 transferring dockerfile: 272B done
#2 [internal] load metadata for docker.io/library/alpine:3.20      ← tag resolved at the registry (step 3)
#2 DONE 1.9s
#3 [internal] load .dockerignore
#3 transferring context: 2B 0.1s done                              ← no .dockerignore yet (added in Task 4)
#5 [1/4] FROM docker.io/library/alpine:3.20@sha256:d9e853e87e55526f6b2917df91a2115c36dd7c696a35be12163d44e6e2a4b6bc
#4 [internal] load build context
#4 transferring context: 326B 0.0s done                            ← build context sent (step 1)
#5 sha256:25f1d6b1951a... 3.63MB / 3.63MB 0.3s done                ← missing base layer downloaded (step 3)
#5 extracting sha256:25f1d6b1951a... 0.1s done
#6 [2/4] WORKDIR /app                                              ← one node per instruction (steps 4–5)
#6 DONE 0.1s
#7 [3/4] COPY Directory.Build.props global.json ./
#7 DONE 0.1s
#8 [4/4] RUN wc -c Directory.Build.props global.json > sizes.txt
#8 DONE 0.4s
#9 exporting to image                                              ← manifest + config assembled (step 7)
#9 exporting layers 0.0s done
#9 writing image sha256:a304707936c9f9a323b835d0fd089feaae7f2638efd59c75272d257afe9e45dd done
#9 naming to docker.io/library/anatomy-demo done                   ← tag applied, loaded into local store (step 8)
```

The same command run a second time, with no change:

```text
#4 [1/4] FROM docker.io/library/alpine:3.20@sha256:d9e8...
#5 transferring context: 73B 0.0s done
#6 [2/4] WORKDIR /app
#6 CACHED
#7 [3/4] COPY Directory.Build.props global.json ./
#7 CACHED
#8 [4/4] RUN wc -c Directory.Build.props global.json > sizes.txt
#8 CACHED
#9 writing image sha256:a304707936c9... done                        ← identical image ID
```

Observations:

- The tag `alpine:3.20` was resolved to a **digest** (`@sha256:d9e8...`). That is what the build really uses.
- Only the instructions that touch the filesystem (`WORKDIR`, `COPY`, `RUN`) appear as `[n/4]` steps. `ARG`, `ENV`, `EXPOSE`, `USER`, `ENTRYPOINT` and `CMD` have no step in the output: they are only written into the image config during `exporting to image`.
- The context transfer was only **326 B**, although `dotnet-app/` contains `bin/`, `obj/` and `publish/` from Task 1. BuildKit only sends the paths that the Dockerfile actually needs (here the two `COPY` sources), and on a repeated build only what changed (73 B). With `COPY . .` (Task 3), the whole folder has to be sent, which makes `.dockerignore` matter (Task 4).

### Layers vs metadata (evidence)

```text
$ docker history anatomy-demo
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
a304707936c9   12 seconds ago   CMD ["sizes.txt"]                               0B        buildkit.dockerfile.v0
<missing>      12 seconds ago   ENTRYPOINT ["cat"]                              0B        buildkit.dockerfile.v0
<missing>      12 seconds ago   USER nobody                                     0B        buildkit.dockerfile.v0
<missing>      12 seconds ago   EXPOSE [8080/tcp]                               0B        buildkit.dockerfile.v0
<missing>      12 seconds ago   ENV APP_ENV=lab                                 0B        buildkit.dockerfile.v0
<missing>      12 seconds ago   ARG BUILD_NOTE=demo                             0B        buildkit.dockerfile.v0
<missing>      12 seconds ago   RUN /bin/sh -c wc -c Directory.Build.props g…   70B       buildkit.dockerfile.v0
<missing>      12 seconds ago   COPY Directory.Build.props global.json ./ # …   236B      buildkit.dockerfile.v0
<missing>      12 seconds ago   WORKDIR /app                                    0B        buildkit.dockerfile.v0
<missing>      6 months ago     CMD ["/bin/sh"]                                 0B        buildkit.dockerfile.v0
<missing>      6 months ago     ADD alpine-minirootfs-3.20.10-x86_64.tar.gz …   7.81MB    buildkit.dockerfile.v0

$ docker image inspect anatomy-demo --format '{{json .RootFS.Layers}}'     # 4 filesystem layers
["sha256:08bc4e53...", "sha256:94683cce...", "sha256:58b01f54...", "sha256:15ad8f62..."]
#  = alpine base, WORKDIR /app (creates the directory), COPY, RUN

$ docker image inspect anatomy-demo --format 'Entrypoint={{json .Config.Entrypoint}} Cmd={{json .Config.Cmd}} User={{json .Config.User}} WorkingDir={{json .Config.WorkingDir}} Env={{json .Config.Env}} ExposedPorts={{json .Config.ExposedPorts}}'
Entrypoint=["cat"] Cmd=["sizes.txt"] User="nobody" WorkingDir="/app" Env=["PATH=...","APP_ENV=lab"] ExposedPorts={"8080/tcp":{}}
```

`docker images` reports **7.81 MB**: the alpine layer plus 236 B + 70 B. Every metadata instruction is 0 B. They do not exist as layers; they live in `.Config`. `WORKDIR` is listed as 0 B but still produced a (nearly empty) filesystem layer, because it created the `/app` directory. That matches the lab's "metadata (plus the directory)".

**A metadata-only change does not touch the layers.** In a copy of the Dockerfile, only `ENV APP_ENV=lab` was changed to `lab2`. All filesystem steps were `CACHED`, and the result has **the same four layer digests** but a **different image ID**, because the image ID is the hash of the config, and the config changed:

```text
anatomy-demo:ctx  id=sha256:a42e943b59ee...  layers=[08bc4e53…, 94683cce…, 9b510fce…, a5c96ccb…]
anatomy-demo:ctx2 id=sha256:2a517506d910...  layers=[08bc4e53…, 94683cce…, 9b510fce…, a5c96ccb…]   (ENV changed)
```

**A file change invalidates its layer and everything after it (Rule 1).** In a scratch copy of the context, `global.json` was edited (`latestFeature` → `latestPatch`) and the image rebuilt:

```text
#6 [2/4] WORKDIR /app
#6 CACHED                                                  ← before the change: reused
#7 [3/4] COPY Directory.Build.props global.json ./
#7 DONE 0.0s                                               ← checksum of copied files changed → cache miss
#8 [4/4] RUN wc -c Directory.Build.props global.json > sizes.txt
#8 DONE 0.3s                                               ← RUN text unchanged, but parent layer is new → rebuilt
```

### Runtime behavior (ENTRYPOINT/CMD, ENV/ARG)

```text
$ docker run --rm anatomy-demo                       # ENTRYPOINT ["cat"] + CMD ["sizes.txt"]
      149 Directory.Build.props
       87 global.json
      236 total

$ docker run --rm anatomy-demo /etc/alpine-release   # arguments replace CMD, ENTRYPOINT stays
3.20.10

$ docker run --rm --entrypoint sh anatomy-demo -c 'echo "APP_ENV=$APP_ENV BUILD_NOTE=${BUILD_NOTE:-<unset>} user=$(id -un)"'
APP_ENV=lab BUILD_NOTE=<unset> user=nobody
```

### Answers

**1. Which instructions produce layers with files, and which only change metadata? Why does it matter for the size?**

`FROM` (brings in the base image's layers), `COPY`/`ADD` and `RUN` create layers with files. `WORKDIR` creates only an empty directory if it does not exist yet. `ENV`, `ARG`, `LABEL`, `EXPOSE`, `USER`, `ENTRYPOINT`, `CMD` and `HEALTHCHECK` only change the image configuration. The `docker history` above shows them all with 0 B, and the config change kept the same layer digests.

It matters because the image size is the **sum of its file layers**, and layers are immutable. A file added in one layer and deleted in a later `RUN` is still stored in the earlier layer; the deletion only hides it. The size is therefore decided by `FROM` (which base image) and by what `COPY`/`RUN` put in. For this lab:

- The .NET build output is only 916 KB (Task 1), but an image based on `mcr.microsoft.com/dotnet/sdk:10.0` carries the whole SDK.
- `COPY . .` also copies `bin/`, `obj/` and `publish/` into a layer.

Adding `ENV` or `LABEL` lines, on the other hand, costs nothing. Shrinking an image means choosing a smaller `FROM` and copying fewer files: a multi-stage build (Task 5) and `.dockerignore` (Task 4).

**2. Why does Rule 1 make the order of instructions important?**

The cache key of a step includes its **parent layer**. When one step misses the cache, every later step has a new parent, so each of them misses too, even if its own text and inputs did not change. The experiment shows it: only `global.json` changed, yet the `RUN` after the `COPY` was executed again.

The cache can therefore only be reused up to the **first** changed step, and the order decides how much work a typical change costs. If an expensive step (`dotnet restore` downloading NuGet packages, `mvn dependency:go-offline` downloading the Maven repository) comes **after** a `COPY` of frequently changing files (`COPY . .` with `Program.cs`), every code edit repeats the download. If it comes **after a `COPY` of only the dependency files** (`DemoApi.csproj`, `pom.xml`), a code edit leaves it cached and only the compile step runs again. This is the change from `Dockerfile.naive` to `Dockerfile.cache` in Task 4.

**3. `ENTRYPOINT` vs `CMD`, and `ENV` vs `ARG`?**

- **`ENTRYPOINT`** is the program that the container always runs. **`CMD`** holds its *default arguments* (or the default command, if there is no `ENTRYPOINT`). Arguments after the image name in `docker run` **replace `CMD`** but keep the `ENTRYPOINT`. Above, `docker run anatomy-demo /etc/alpine-release` still ran `cat`, now on a different file. Replacing the `ENTRYPOINT` needs an explicit `--entrypoint`. For the lab apps, `ENTRYPOINT ["dotnet", "DemoApi.dll"]` makes the container *be* the API. Both should use the exec form (JSON array), so that the app runs as PID 1 and receives `SIGTERM`.
- **`ARG`** exists **only during the build**. Its value is set with `--build-arg`, and it is scoped to the stage in which it is declared. **`ENV`** is stored in the image config and is present **in every running container** (and in later build steps). Above, `APP_ENV=lab` was set at runtime and `BUILD_NOTE` was not. For the lab apps, `ENV ASPNETCORE_HTTP_PORTS=8080` configures the running API, while `ARG APP_VERSION` (Task 6) only feeds the build.

  **Warning:** `docker history` shows the `ARG` line (`ARG BUILD_NOTE=demo`). Build arguments are therefore *not* secret. Task 6.5 demonstrates the leak.

The experiment images (`anatomy-demo:*`) were deleted after the observations.

## Task 3 – Naive Dockerfiles

Both Dockerfiles are exactly the ones from the lab and are kept unchanged as the baseline: [`dotnet-app/Dockerfile.naive`](https://github.com/ceciliagalvaoo/docker-build-demo/blob/main/dotnet-app/Dockerfile.naive) and [`java-app/Dockerfile.naive`](https://github.com/ceciliagalvaoo/docker-build-demo/blob/main/java-app/Dockerfile.naive).

```dockerfile
# dotnet-app/Dockerfile.naive
FROM mcr.microsoft.com/dotnet/sdk:10.0
WORKDIR /app
COPY . .
RUN dotnet publish src/DemoApi/DemoApi.csproj -c Release -o out
ENV ASPNETCORE_HTTP_PORTS=8080
ENTRYPOINT ["dotnet", "out/DemoApi.dll"]
```

```dockerfile
# java-app/Dockerfile.naive
FROM maven:3-eclipse-temurin-21
WORKDIR /app
COPY . .
RUN mvn -B -ntp package -DskipTests
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "target/demo-0.0.1-SNAPSHOT.jar"]
```

Conditions: Docker Engine 28.5.1 / BuildKit v0.25.1 on WSL2. Both base images were pulled during these builds, so this is a cold build. There is no `.dockerignore` yet, and the working folders still contain the outputs of the Task 1 native builds (`dotnet-app`: `bin/` + `obj/` + `publish/` ≈ 16 MB; `java-app/target/` = 20 MB). That is the realistic situation of a developer machine.

### 3.1 .NET – build and run

```text
$ cd dotnet-app
$ docker build --progress=plain -f Dockerfile.naive -t demo-dotnet:naive .
#2 [internal] load metadata for mcr.microsoft.com/dotnet/sdk:10.0
#4 [1/4] FROM mcr.microsoft.com/dotnet/sdk:10.0@sha256:de4fe90a98845d75910b9769662465c84de1a934665c431279d1382b2bddc02b
#5 [internal] load build context
#5 transferring context: 15.99MB 4.1s done
#4 DONE 24.1s                                          ← SDK image download + extract
#6 [2/4] WORKDIR /app
#7 [3/4] COPY . .
#8 [4/4] RUN dotnet publish src/DemoApi/DemoApi.csproj -c Release -o out
#8 3.266   Restored /app/src/DemoApi/DemoApi.csproj (in 1.57 sec).
#8 7.130   DemoApi -> /app/src/DemoApi/bin/Release/net10.0/DemoApi.dll
#8 7.194   DemoApi -> /app/out/
#8 DONE 7.3s
#9 naming to docker.io/library/demo-dotnet:naive done
exit=0, wall time 33.9 s

$ docker run --rm -p 8080:8080 demo-dotnet:naive
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://[::]:8080
      Hosting environment: Production
      Content root path: /app

$ curl http://localhost:8080/weatherforecast
[{"date":"2026-11-05","temperatureC":-19,"summary":"Scorching","temperatureF":-2},{"date":"2026-11-06","temperatureC":22,"summary":"Bracing","temperatureF":71}, ...]
HTTP 200
```

The SDK inside the image is 10.0.403 (`docker exec ... dotnet --list-sdks`), so the `global.json` from Task 1 (10.0.100, `latestFeature`) is satisfied.

### 3.2 Java – build and run

```text
$ cd java-app
$ docker build --progress=plain -f Dockerfile.naive -t demo-java:naive .
#4 [1/4] FROM docker.io/library/maven:3-eclipse-temurin-21@sha256:9673eef9ed503146591667ffb92eaaa22f1b8899ed60d9df0df4da3f3a1214a6
#5 transferring context: 20.02MB 1.3s done
#4 DONE 14.5s
#7 [3/4] COPY . .
#8 [4/4] RUN mvn -B -ntp package -DskipTests
#8 15.98 [INFO] Building jar: /app/target/demo-0.0.1-SNAPSHOT.jar
#8 17.52 [INFO] BUILD SUCCESS
#8 17.52 [INFO] Total time:  15.790 s
#8 DONE 17.7s
#9 naming to docker.io/library/demo-java:naive done
exit=0, wall time 37.7 s

$ docker run --rm -p 8080:8080 demo-java:naive
 :: Spring Boot ::                (v4.1.2)
... Tomcat started on port 8080 (http) with context path '/'
... Started DemoApplication in 2.177 seconds (process running for 2.776)

$ curl http://localhost:8080/
{"timestamp":"2026-11-04T16:08:13.776Z","status":404,"error":"Not Found","path":"/"}
HTTP 404        ← Spring Boot's error handler answers: the app is up (the template has no controller, see Task 1)
```

The image runs Maven 3.10.0 and `javac 21.0.13` (Temurin JDK 21).

### 3.3 Inspection

```text
$ docker images
REPOSITORY    TAG     IMAGE ID       SIZE
demo-java     naive   cb7a5fc968f4   669MB
demo-dotnet   naive   362d3c9b23b1   940MB
```

| Image | Size (`docker images`) | Exact size (`.Size`) | Filesystem layers (`RootFS.Layers`) | Our own layers |
|-------|------------------------|----------------------|-------------------------------------|----------------|
| `demo-dotnet:naive` | **940 MB** | 939,873,285 B | 14 | `COPY . .` 15.9 MB + `RUN dotnet publish` 6.2 MB |
| `demo-java:naive` | **669 MB** | 668,596,654 B | 13 | `COPY . .` 20 MB + `RUN mvn package` 83.8 MB |

For comparison: the deployable output itself is the 916 KB `publish/` folder and the 20 MB JAR (Task 1).

#### `docker history demo-dotnet:naive`

```text
IMAGE          CREATED              CREATED BY                                      SIZE      COMMENT
362d3c9b23b1   About a minute ago   ENTRYPOINT ["dotnet" "out/DemoApi.dll"]         0B        buildkit.dockerfile.v0
<missing>      About a minute ago   ENV ASPNETCORE_HTTP_PORTS=8080                  0B        buildkit.dockerfile.v0
<missing>      About a minute ago   RUN /bin/sh -c dotnet publish src/DemoApi/De…   6.2MB     buildkit.dockerfile.v0
<missing>      About a minute ago   COPY . . # buildkit                             15.9MB    buildkit.dockerfile.v0
<missing>      About a minute ago   WORKDIR /app                                    0B        buildkit.dockerfile.v0
<missing>      9 days ago           RUN /bin/sh -c powershell_version=7.6.8     …   60.9MB    buildkit.dockerfile.v0
<missing>      9 days ago           RUN /bin/sh -c ln -s /usr/share/dotnet/dnx /…   21B       buildkit.dockerfile.v0
<missing>      9 days ago           COPY /PowerShell.Linux.x64.7.6.8.nupkg / # b…   21.1MB    buildkit.dockerfile.v0
<missing>      9 days ago           COPY /dotnet /usr/share/dotnet # buildkit       531MB     buildkit.dockerfile.v0
<missing>      9 days ago           RUN /bin/sh -c apt-get update     && apt-get…   74.9MB    buildkit.dockerfile.v0
<missing>      9 days ago           ENV DOTNET_GENERATE_ASPNET_CERTIFICATE=false…   0B        buildkit.dockerfile.v0
<missing>      9 days ago           COPY /dotnet /usr/share/dotnet # buildkit       27.1MB    buildkit.dockerfile.v0
<missing>      9 days ago           ENV ASPNET_VERSION=10.0.14                      0B        buildkit.dockerfile.v0
<missing>      9 days ago           RUN /bin/sh -c ln -s /usr/share/dotnet/dotne…   24B       buildkit.dockerfile.v0
<missing>      9 days ago           COPY /dotnet /usr/share/dotnet # buildkit       82.8MB    buildkit.dockerfile.v0
<missing>      9 days ago           ENV DOTNET_VERSION=10.0.14                      0B        buildkit.dockerfile.v0
<missing>      9 days ago           RUN /bin/sh -c groupadd         --gid=$APP_U…   9.64kB    buildkit.dockerfile.v0
<missing>      9 days ago           RUN /bin/sh -c apt-get update     && apt-get…   42MB      buildkit.dockerfile.v0
<missing>      9 days ago           ENV APP_UID=1654 ASPNETCORE_HTTP_PORTS=8080 …   0B        buildkit.dockerfile.v0
<missing>      10 days ago          /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B
<missing>      10 days ago          /bin/sh -c #(nop) ADD file:eed88e94d6aa7c9da…   78.2MB
<missing>      10 days ago          /bin/sh -c #(nop)  LABEL org.opencontainers.…   0B
<missing>      10 days ago          /bin/sh -c #(nop)  ARG RELEASE                  0B
...
```

#### `docker history demo-java:naive`

```text
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
cb7a5fc968f4   19 seconds ago   ENTRYPOINT ["java" "-jar" "target/demo-0.0.1…   0B        buildkit.dockerfile.v0
<missing>      19 seconds ago   EXPOSE [8080/tcp]                               0B        buildkit.dockerfile.v0
<missing>      19 seconds ago   RUN /bin/sh -c mvn -B -ntp package -DskipTes…   83.8MB    buildkit.dockerfile.v0
<missing>      37 seconds ago   COPY . . # buildkit                             20MB      buildkit.dockerfile.v0
<missing>      37 seconds ago   WORKDIR /app                                    0B        buildkit.dockerfile.v0
<missing>      17 hours ago     CMD ["mvn"]                                     0B        buildkit.dockerfile.v0
<missing>      17 hours ago     ENTRYPOINT ["/usr/local/bin/mvn-entrypoint.s…   0B        buildkit.dockerfile.v0
<missing>      17 hours ago     ENV MAVEN_CONFIG=/root/.m2                      0B        buildkit.dockerfile.v0
<missing>      17 hours ago     COPY /usr/share/maven /usr/share/maven # bui…   11.8MB    buildkit.dockerfile.v0
<missing>      17 hours ago     RUN /bin/sh -c apt-get update   && apt-get i…   76.3MB    buildkit.dockerfile.v0
<missing>      5 days ago       COPY --chmod=755 entrypoint.sh /__cacert_ent…   5.98kB    buildkit.dockerfile.v0
<missing>      5 days ago       RUN /bin/sh -c set -eux;     ARCH="$(dpkg --…   308MB     buildkit.dockerfile.v0
<missing>      5 days ago       ENV JAVA_VERSION=jdk-21.0.13+9                  0B        buildkit.dockerfile.v0
<missing>      5 days ago       RUN /bin/sh -c set -eux;     apt-get update;…   67MB      buildkit.dockerfile.v0
<missing>      10 days ago      umoci raw add-layer --image /home/buildd/roc…   102MB
...  (metadata-only rows of the base image omitted: LABEL/ENV/umoci config, all 0B)
```

#### Layers, entrypoint and user

```text
$ docker image inspect demo-dotnet:naive --format '{{json .RootFS.Layers}}'
["sha256:bec87777bc2f...","sha256:fa09e76fa81b...","sha256:7be251a39e9c...","sha256:2c16277b36ff...",
 "sha256:361fe91f35cc...","sha256:eaa51743b39d...","sha256:d81312fed0de...","sha256:b6fd3ac23e41...",
 "sha256:af2d9ffb5214...","sha256:a4bb267a5014...","sha256:e8cce59db267...","sha256:f01b1409e595...",
 "sha256:aa7f149967f7...","sha256:5301209a9dac..."]                       # 14 layers

$ docker image inspect demo-dotnet:naive --format '{{json .Config.Entrypoint}} {{json .Config.User}}'
["dotnet","out/DemoApi.dll"] ""

$ docker image inspect demo-java:naive --format '{{json .Config.Entrypoint}} {{json .Config.User}}'
["java","-jar","target/demo-0.0.1-SNAPSHOT.jar"] ""
```

Inside the running containers (`docker exec`):

```text
demo-dotnet:naive                                demo-java:naive
uid=0(root) gid=0(root) groups=0(root)           uid=0(root) gid=0(root) groups=0(root)
USER   PID COMMAND                               USER   PID COMMAND
root     1 dotnet out/DemoApi.dll                root     1 java -jar target/demo-0.0.1-SNAPSHOT.jar

/app (18 MB): DemoApp.slnx Directory.Build.props  /app (20 MB): .gitattributes .gitignore .mvn
  Dockerfile.naive global.json out publish src      Dockerfile.naive HELP.md mvnw mvnw.cmd pom.xml
  tests                                             src target
/root/.nuget: 3.0 MB                              /root/.m2: 69 MB
dotnet --list-sdks → 10.0.403                     mvn -v → Apache Maven 3.10.0; javac 21.0.13
```

### Answers

**Which layer is the largest? Why does a runtime image contain the whole SDK, the source code and the `obj/` folders?**

The largest layers belong to the **base image**, not to our application:

- **.NET:** `COPY /dotnet /usr/share/dotnet` (**531 MB**, the .NET SDK), followed by the Ubuntu base (78.2 MB), the apt packages (74.9 MB + 42 MB) and PowerShell (60.9 MB + 21.1 MB). Our own two layers add only 22 MB to 940 MB.
- **Java:** the JDK layer `RUN ... ARCH="$(dpkg ...` (**308 MB**), followed by the Ubuntu base (102 MB) and the apt packages (76.3 MB + 67 MB). Among our own layers, `RUN mvn package` (83.8 MB) is the largest: it contains the whole Maven local repository `/root/.m2` (69 MB of downloaded plugins and dependencies) plus the rebuilt `target/`.

The runtime image contains all of this because a single-stage Dockerfile **has only one image**: the one used to build is the one that is shipped. Everything that `FROM` brought in for building (SDK/JDK, Maven, compilers, PowerShell) and everything that `COPY . .` and `RUN` put on disk stays in the layers:

- `COPY . .` copied the whole build context: the source code, `tests/`, `Dockerfile.naive`, and also the local `bin/`, `obj/` and `publish/` folders from Task 1 (15.9 MB), because there is no `.dockerignore`.
- `dotnet publish` then created a new `obj/` and `bin/` inside the image and filled the NuGet cache.

Nothing is ever removed, and even a `rm` in a later step would not shrink the earlier layers.

**Which user does the application run as?**

`root` (uid 0) in both images. `.Config.User` is empty (`""`), which means root, and `docker exec ... id` / `ps` confirm that PID 1 (`dotnet` / `java`) runs as `root`. The SDK image even defines a non-root user `app` (`APP_UID=1654`, visible in `docker history`), but the naive Dockerfile never switches to it.

**At least three problems with these images:**

1. **Size:** 940 MB and 669 MB to ship a 916 KB folder and a 20 MB JAR. Every pull, push and deploy moves hundreds of MB, and the registry and nodes store them.
2. **Attack surface:** the images ship a full SDK/JDK, a compiler, Maven, PowerShell, `apt`, a shell and many OS packages. An attacker who gets into the container has a complete toolchain, and every one of these packages adds CVEs to scan and patch.
3. **Runs as root:** a vulnerability in the app gives root inside the container, which makes a container escape or tampering with the filesystem much easier.
4. **Source code and build leftovers in the image:** the source, tests, `Dockerfile.naive`, `obj/` and `bin/` from the developer's machine (even `HELP.md` and `.gitignore`), plus `/root/.nuget` and `/root/.m2`. Any local file, for example a `.env` with secrets, would be copied the same way.
5. **Slow rebuilds:** `COPY . .` comes before the restore/package step. Any change to any file, even a comment or a README, invalidates the `COPY` layer, and by Rule 1 the dependency download (NuGet restore, the 69 MB Maven repository) runs again. A large context (16–20 MB without `.dockerignore`) is also sent on every build. Task 4 measures this.
6. **Not reproducible:** the result depends on what is lying in the local folder. The Java build runs `mvn package` without `clean` on top of the copied local `target/`. The base images are referenced by moving tags (`sdk:10.0`, `maven:3-eclipse-temurin-21`).
7. **No tests:** `-DskipTests` skips the tests, and the .NET Dockerfile never runs `dotnet test` (addressed in Task 6).

## Task 4 – Layer cache and `.dockerignore`

Environment: Docker Engine 28.5.1 / BuildKit v0.25.1 on WSL2, branch `task-4-cache-dockerignore`. All temporary edits below (one character in `Program.cs`, an extra NuGet package in `DemoApi.csproj`, other base image tags) were made only for measuring. Afterwards they were reverted from a byte-exact backup, verified with `sha256sum -c`. The base-image experiments used temporary copies of `Dockerfile.cache` outside the repository, so the committed files contain none of these changes.

### 4.1 Observing (and breaking) the cache with the naive Dockerfile

**Rebuild without editing anything.** The first rebuild in this task was, surprisingly, **not** fully cached:

```text
$ docker build --progress=plain -f Dockerfile.naive -t demo-dotnet:naive .
#5 transferring context: 35.14kB 4.8s done
#6 [2/4] WORKDIR /app
#6 CACHED
#7 [3/4] COPY . .                                             ← executed, not CACHED
#8 [4/4] RUN dotnet publish src/DemoApi/DemoApi.csproj ...    ← executed, not CACHED
[wall 12.95s]
```

Investigation: `file Dockerfile.naive` reported `CRLF line terminators`, with a modification time of 17:11:19, which is exactly when `git switch main && git pull` ran with Windows Git (`core.autocrlf=true`). The checkout had rewritten `Dockerfile.naive` (committed in Task 3 with LF) with CRLF endings. Without a `.dockerignore`, `Dockerfile.naive` is part of the context and is copied by `COPY . .`, so its new checksum invalidated the `COPY` layer and, by Rule 1, the `RUN` after it. This is a real example of a file that has **nothing to do with the application** breaking the cache, which the `Dockerfile*` line of `.dockerignore` (4.2) prevents.

Running the same command again, now without any change:

```text
#6 [2/4] WORKDIR /app
#6 CACHED
#7 [3/4] COPY . .
#7 CACHED
#8 [4/4] RUN dotnet publish src/DemoApi/DemoApi.csproj -c Release -o out
#8 CACHED
```

**Edit one character in `src/DemoApi/Program.cs`** (`// Add services to the container.` → `...container!`, a comment):

```text
$ docker build --progress=plain -f Dockerfile.naive -t demo-dotnet:naive .
#6 [2/4] WORKDIR /app
#6 CACHED
#7 [3/4] COPY . .
#7 DONE 0.1s                                                   ← rebuilt
#8 [4/4] RUN dotnet publish src/DemoApi/DemoApi.csproj -c Release -o out
#8 3.075   Restored /app/src/DemoApi/DemoApi.csproj (in 1.59 sec).   ← NuGet restore repeated
#8 6.284   DemoApi -> /app/src/DemoApi/bin/Release/net10.0/DemoApi.dll
#8 6.356   DemoApi -> /app/out/
#8 DONE 6.5s                                                   ← rebuilt
[wall 12.97s]
```

**Which steps were rebuilt?** `COPY . .` and `RUN dotnet publish`. Only `FROM` and `WORKDIR` were reused.

**Was the NuGet restore repeated? Why?** Yes (`Restored ... (in 1.59 sec)`). `dotnet publish` restores implicitly, and the restore happens in the same `RUN` layer as the compilation. That layer comes after `COPY . .`, whose cache key contains the checksum of **every** file in the context. A comment in `Program.cs` changes that checksum, so the `COPY` layer misses the cache, and by Rule 1 so does the `RUN`, including the dependency download that has not changed at all. With only one small package, the restore costs about 1.6 s here. In a real project with many packages, every code edit would pay for the complete download again.

### 4.2 `.dockerignore`

Both files were created exactly as in the lab: [`dotnet-app/.dockerignore`](https://github.com/ceciliagalvaoo/docker-build-demo/blob/main/dotnet-app/.dockerignore) and [`java-app/.dockerignore`](https://github.com/ceciliagalvaoo/docker-build-demo/blob/main/java-app/.dockerignore).

**How it was measured.** BuildKit transfers the context *incrementally*: on a repeated build it only sends what changed (see the 35 kB above). To compare full context sizes, the context-transfer cache of the builder was cleared before each measurement with `docker builder prune -f --filter type=source.local` (this does not remove any layer cache). Then the naive build was run and its `transferring context` line recorded. In addition, the content that actually arrives in the builder was counted with a throwaway Dockerfile (`FROM alpine:3.20` / `COPY . /ctx` / `RUN find /ctx -type f | wc -l; du -sh /ctx`, built with `--no-cache`).

| Context | `transferring context` (full) | Files | Size in builder | Largest parts |
|---------|-------------------------------|-------|-----------------|---------------|
| `dotnet-app/` **without** `.dockerignore` | **15.99 MB** | 334 | 16.4 MB | `tests/` 12.9 MB (test `bin/`), `src/` 2.5 MB (`bin/`, `obj/`), `publish/` 940 KB, `Dockerfile.naive`, `.gitignore` |
| `dotnet-app/` **with** `.dockerignore` | **4.55 kB** | 11 | 68 KB | `src/` 36 KB, `tests/` 16 KB, `DemoApp.slnx`, `Directory.Build.props`, `global.json` |
| `java-app/` **without** `.dockerignore` | **20.02 MB** | 23 | 19.3 MB | `target/` 19.2 MB (fat JAR, classes), `src/` 68 KB, `mvnw*`, `.mvn/` |
| `java-app/` **with** `.dockerignore` | **23.48 kB** | 8 | 116 KB | `src/` 68 KB, `mvnw`, `mvnw.cmd`, `.mvn/`, `pom.xml`, `.gitattributes` |

The context shrank by about **3,500×** for .NET (15.99 MB → 4.55 kB, 334 → 11 files) and about **850×** for Java (20.02 MB → 23.48 kB, 23 → 8 files). What is left is only the source and the build definitions.

Side effects observed:

- The naive images got smaller, because `COPY . .` no longer copies the local build output: `demo-dotnet:naive` **940 MB → 925 MB**, `demo-java:naive` **669 MB → 649 MB**.
- The images are now reproducible from the sources. The local `bin/`, `obj/` and `target/` of the developer's machine can no longer leak into the image, and a local `.env` would be excluded.
- Files that are irrelevant for the build (`Dockerfile*`, `*.md`, `.gitignore`) can no longer invalidate the cache, which is exactly what happened in 4.1 with the CRLF rewrite of `Dockerfile.naive`.

### 4.3 Ordering the instructions for the cache

[`dotnet-app/Dockerfile.cache`](https://github.com/ceciliagalvaoo/docker-build-demo/blob/main/dotnet-app/Dockerfile.cache), exactly as in the lab:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0
WORKDIR /src

# 1. dependency definition only → this layer changes rarely
COPY Directory.Build.props ./
COPY src/DemoApi/DemoApi.csproj src/DemoApi/
RUN dotnet restore src/DemoApi/DemoApi.csproj

# 2. source code → this layer changes often
COPY src/DemoApi/ src/DemoApi/
RUN dotnet publish src/DemoApi/DemoApi.csproj -c Release -o /app --no-restore

ENV ASPNETCORE_HTTP_PORTS=8080
WORKDIR /app
ENTRYPOINT ["dotnet", "DemoApi.dll"]
```

**Naive vs cache-friendly order.** The naive file copies *everything* and then runs one command that restores and compiles. The cache-friendly file copies only the files that define dependencies (`Directory.Build.props`, `DemoApi.csproj`), restores, and only then copies the source and compiles with `--no-restore`. The expensive restore layer now depends only on the dependency definition, not on the code.

Git note: the `.gitignore` generated by `dotnet new gitignore` contains the Visual Studio rule `*.[Cc]ache` (line 236), which matched `Dockerfile.cache`, so the file would silently never have been committed (`git check-ignore -v dotnet-app/Dockerfile.cache` → `dotnet-app/.gitignore:236:*.[Cc]ache`). An exception `!Dockerfile.cache` was added at the end of `dotnet-app/.gitignore`.

Initial build (`docker build -f Dockerfile.cache -t demo-dotnet:cache .`): all 8 steps executed, `Restored ... (in 1.5 sec)`, wall time 9.56 s. `docker run -p 8080:8080 demo-dotnet:cache` + `curl /weatherforecast` → **HTTP 200**. Size **925 MB** (still the full SDK; Task 5 fixes that).

#### Cache table (from the real `--progress=plain` logs)

| Change you make | `COPY ...csproj` | `RUN dotnet restore` | `COPY src/DemoApi/` | `RUN dotnet publish` | Wall time |
|-----------------|------------------|----------------------|---------------------|----------------------|-----------|
| Nothing (rebuild) | CACHED | CACHED | CACHED | CACHED | 1.75 s |
| Edit `Program.cs` (one character in a comment) | CACHED | CACHED | **rebuilt** | **rebuilt** | 5.85 s |
| Add a NuGet package to the `.csproj` (`Humanizer.Core 2.14.1`) | **rebuilt** | **rebuilt** (`Restored ... in 1.8 sec`) | **rebuilt** | **rebuilt** | 9.14 s |
| Change the base image tag `10.0` → `10.0.0` (as in the lab) | – | – | – | – | **build failed** (1.14 s) |
| … → `10.0.403` (another tag, **same digest** as `10.0`) | CACHED | CACHED | CACHED | CACHED | 1.34 s |
| … → `10.0.100` (another tag, **different digest**) | **rebuilt** | **rebuilt** (`Restored ... in 1.48 sec`) | **rebuilt** | **rebuilt** | 36.99 s (25.1 s of it pulling the image) |

`WORKDIR /src` and `COPY Directory.Build.props` behaved like the `.csproj` column. They were rebuilt only in the base-image row.

Notes on the rows:

- **Edit `Program.cs`:** the first two dependency steps were `CACHED`. The restore was *not* repeated: compare 4.1, where the same edit repeated it.
- **Add a NuGet package:** `COPY src/DemoApi/` would have been rebuilt anyway, because the `.csproj` also lives in `src/DemoApi/`. By Rule 1 it is rebuilt in any case, since its parent (the restore layer) is new.
- **`10.0.0`:** the lab's example tag does **not exist** for the SDK image, because SDK tags use SDK versions (`10.0.100` … `10.0.403`):

  ```text
  #2 ERROR: mcr.microsoft.com/dotnet/sdk:10.0.0: not found
  ERROR: failed to build: failed to solve: mcr.microsoft.com/dotnet/sdk:10.0.0: failed to resolve source metadata for mcr.microsoft.com/dotnet/sdk:10.0.0: mcr.microsoft.com/dotnet/sdk:10.0.0: not found
  ```

  The registry's tag list (`https://mcr.microsoft.com/v2/dotnet/sdk/tags/list`) and `docker buildx imagetools inspect` were then used to pick two existing tags:

  ```text
  10.0       Digest: sha256:de4fe90a98845d75910b9769662465c84de1a934665c431279d1382b2bddc02b
  10.0.403   Digest: sha256:de4fe90a98845d75910b9769662465c84de1a934665c431279d1382b2bddc02b
  10.0.100   Digest: sha256:c7445f141c04f1a6b454181bd098dcfa606c61ba0bd213d0a702489e5bd4cd71
  ```

- **`10.0.403`:** everything was `CACHED`. The cache key of `FROM` is the **resolved digest**, not the text of the tag, and `10.0.403` is currently the same image as `10.0`.
- **`10.0.100`:** a different digest, so every later step was rebuilt, after a 25 s download of the other SDK image.

**Conclusion.** The most expensive change is a change of the **base image** (a different digest). It is the first layer, so by Rule 1 every layer is rebuilt: the new base image (hundreds of MB) is downloaded, all dependencies are restored again, and the code is compiled again (37 s here, against 6 s for a code edit). This also happens without editing the Dockerfile at all, when a moving tag such as `10.0` gets a new patch release, which is why pinning the base image by version or digest matters. The next most expensive change is a **change of the dependencies** (`.csproj`/`pom.xml`): restore and everything after it run again (9.1 s). The cheapest change is the most frequent one, a **source code edit**: with `Dockerfile.cache` it only repeats `COPY src` + `publish` (5.9 s), while the restore stays cached. With `Dockerfile.naive`, the same edit also repeated the restore. In this tiny demo, the restore only takes about 1.5–3 s, so the absolute gain is small. The *structure* is what matters, because in a real project the restore and download are often the slowest part of the build.

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
| 2 | Task 2 questions answered (with real build observations) | ✅ Done |
| 3 | Naive Dockerfiles built and analyzed with `docker history` | ✅ Done |
| 4 | Cache behavior observed and the table filled in | ✅ Done (tag `10.0.0` does not exist; documented, tested with `10.0.403` and `10.0.100` instead) |
| 4 | `.dockerignore` files created | ✅ Done (context 15.99 MB → 4.55 kB and 20.02 MB → 23.48 kB) |
| 5 | Multi-stage Dockerfiles for .NET and Java, running as non-root | ⏳ Pending |
| 5 | Image comparison table filled in with own measurements | ⏳ Pending |
| 6 | `ARG`, `LABEL`, a `test` stage and a cache mount used | ⏳ Pending |
| 6 | Secret leak demonstrated, and a secret mount used | ⏳ Pending |
| 6 | Multi-platform build | ⏳ Optional |
| 7 | Image tagged, pushed to `ghcr.io`, pulled and run, also by digest | ⏳ Pending |
| 8 | Project containerized and documented | ➖ Out of scope for this report (separate deliverable in the project repository) |
