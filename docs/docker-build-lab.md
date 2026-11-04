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

_Pending._

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
| 1 | .NET and Java apps built with their native tools; build steps listed | ⏳ Pending |
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
