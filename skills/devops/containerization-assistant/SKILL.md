---
name: containerization-assistant
description: Generates hardened, multi-stage Dockerfiles with non-root users, minimal base images, and a .dockerignore, after auto-detecting the application stack. Use when containerizing an application for the first time, when the user asks for a Dockerfile, when migrating from a VM deployment, or when an existing Dockerfile runs as root, uses a fat base image, or leaks build tooling into the runtime layer.
license: Apache-2.0
metadata:
  category: "devops"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "ci-pipeline-synthesizer, k8s-manifest-generator"
---

# Containerization Assistant

Produce a Dockerfile that is minimal, reproducible, runs as non-root, and separates build tooling from the runtime image. Then produce a `.dockerignore` that keeps the build context small.

## Step 1 — Detect the stack

Check the repo root for these markers, in order. First match wins.

| Marker file(s)                                   | Stack          | Builder stage base        | Runtime stage base                              |
| ------------------------------------------------ | -------------- | ------------------------- | ----------------------------------------------- |
| `package.json` + `tsconfig.json`                 | Node (TS)      | `node:<ver>-slim`         | `gcr.io/distroless/nodejs<ver>-debian12`        |
| `package.json` only                              | Node (JS)      | `node:<ver>-slim`         | `gcr.io/distroless/nodejs<ver>-debian12`        |
| `go.mod`                                         | Go             | `golang:<ver>-alpine`     | `gcr.io/distroless/static-debian12` (or `scratch` if `CGO_ENABLED=0`) |
| `Cargo.toml`                                     | Rust           | `rust:<ver>-slim`         | `gcr.io/distroless/cc-debian12`                 |
| `pom.xml` / `build.gradle`                       | Java (JVM)     | `eclipse-temurin:<ver>-jdk` | `eclipse-temurin:<ver>-jre` (or distroless java) |
| `pyproject.toml` / `requirements.txt` / `Pipfile`| Python         | `python:<ver>-slim`       | `python:<ver>-slim` (distroless python is viable if no C extensions) |
| `*.csproj` / `*.sln`                             | .NET           | `mcr.microsoft.com/dotnet/sdk:<ver>` | `mcr.microsoft.com/dotnet/aspnet:<ver>` |
| `Gemfile`                                        | Ruby           | `ruby:<ver>-slim`         | `ruby:<ver>-slim`                               |

Pull `<ver>` from the version pin in the marker file (`engines.node`, `go 1.x`, `python_requires`, etc.). If no pin exists, ask the user — never guess a major version.

If *no* marker matches, stop and ask: "What runtime does this application use?"

## Step 2 — Detect native dependencies

Before choosing the runtime base, scan for packages that compile C extensions or link system libraries:

- **Python:** `psycopg2` (not `-binary`), `cryptography`, `pillow`, `numpy`, `lxml`, `grpcio`
- **Node:** `sharp`, `canvas`, `sqlite3`, `bcrypt` (not `bcryptjs`), anything with `node-gyp` in its install log
- **Ruby:** `nokogiri`, `pg`, `ffi`

If found: the runtime stage needs `libc`/`libstdc++`/specific `.so` files. Switch from `distroless/static` → `distroless/cc`, or from `scratch` → `alpine` + `apk add` the runtime libs, or stay on `-slim` with the specific `apt-get install --no-install-recommends <libs>`.

## Step 3 — Generate the Dockerfile

Always multi-stage. Builder stage has compilers and dev deps; runtime stage has the binary/bundle and nothing else.

### Template (Node/TypeScript)

```dockerfile
# syntax=docker/dockerfile:1.6

# ── Build stage ───────────────────────────────────────────
FROM node:20.11-slim@sha256:<digest> AS builder
WORKDIR /app

# Deps layer — cache hits when only src changes
COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build
RUN npm prune --omit=dev

# ── Runtime stage ─────────────────────────────────────────
FROM gcr.io/distroless/nodejs20-debian12@sha256:<digest>
WORKDIR /app
ENV NODE_ENV=production

COPY --from=builder --chown=nonroot:nonroot /app/dist ./dist
COPY --from=builder --chown=nonroot:nonroot /app/node_modules ./node_modules
COPY --from=builder --chown=nonroot:nonroot /app/package.json ./

USER nonroot
EXPOSE 3000
CMD ["dist/server.js"]
```

### Template (Go, static binary)

```dockerfile
# syntax=docker/dockerfile:1.6

FROM golang:1.22-alpine@sha256:<digest> AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags='-s -w' -o /out/app ./cmd/server

FROM gcr.io/distroless/static-debian12@sha256:<digest>
COPY --from=builder /out/app /app
USER nonroot
EXPOSE 8080
ENTRYPOINT ["/app"]
```

### Template (Python)

```dockerfile
# syntax=docker/dockerfile:1.6

FROM python:3.12-slim@sha256:<digest> AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim@sha256:<digest>
RUN groupadd -r app && useradd -r -g app app
WORKDIR /app
COPY --from=builder /install /usr/local
COPY --chown=app:app . .
USER app
EXPOSE 8000
CMD ["python", "-m", "myapp"]
```

### Non-negotiable properties of the output

1. **Pin the base by digest**, not just tag: `FROM node:20-slim@sha256:abc...`. Get the digest with `docker buildx imagetools inspect node:20-slim`. If the user can't provide digests at generation time, pin the tag and add a `# TODO: pin digest` comment — never leave it at `latest`.
2. **Non-root user.** Distroless ships `nonroot` (uid 65532). For other bases, create one: `RUN useradd -r -u 10001 app` → `USER app`.
3. **Dependency layer before source layer.** `COPY package.json package-lock.json ./` → `RUN npm ci` → `COPY . .`. This preserves the cache when only source changes.
4. **Deterministic install.** `npm ci` not `npm install`. `pip install -r requirements.txt` with a lockfile. `go mod download` before `go build`.
5. **No secrets in any layer.** No `ENV API_KEY=...`, no `COPY .env`. If a secret is needed at build time, use BuildKit mounts: `RUN --mount=type=secret,id=npmrc npm ci`.
6. **HEALTHCHECK only if it won't give a false signal.** `HEALTHCHECK CMD curl -f http://localhost:3000/health || exit 1` requires `curl` in the image — which distroless doesn't have. Either ship a tiny static healthcheck binary or rely on orchestrator probes (K8s liveness) instead.

## Step 4 — Generate .dockerignore

If absent, always generate. Default:

```
.git
.gitignore
**/.DS_Store
**/node_modules
**/__pycache__
**/*.pyc
**/target
**/.venv
**/dist
**/build
**/coverage
.env
.env.*
**/*.log
Dockerfile*
.dockerignore
README*
*.md
```

Then **add back** with `!` any build-time paths the Dockerfile actually `COPY`s. Example: if the Dockerfile does `COPY dist ./dist` (pre-built outside Docker), you need `!dist` at the end.

## Step 5 — Sanity checks before handing off

- [ ] `docker build .` succeeds with no warnings about missing `COPY` sources
- [ ] `docker run --rm <img> id -u` prints a non-zero UID
- [ ] `docker history <img>` shows no layer containing `.git`, `.env`, or `node_modules` in a non-Node image
- [ ] `docker image ls` — final image < 200 MB for Go/Rust, < 400 MB for Node/Python. If larger, something leaked from the builder.
- [ ] No `apt`, `apk`, `pip`, `npm`, `gcc` in the *runtime* stage (run `docker run --rm <img> which apt` — should fail)

## Edge cases

- **Monorepo:** User must specify *which* package to containerize. If the package depends on workspace siblings, the build context is the monorepo root and you need `COPY` filters or a `.dockerignore` negation for just the needed packages. Consider turborepo/nx `prune` if available.
- **App reads local files at runtime (templates, migrations, static assets):** These must be `COPY`d to the runtime stage or it will crash on start. Grep for `readFile`, `open(`, `__dirname`, `os.path.join(os.path.dirname(__file__)` to find them.
- **Needs to write to disk at runtime (uploads, temp files, SQLite):** Non-root can't write to `/app`. Mount a volume, or `mkdir /data && chown app:app /data` in the runtime stage and point the app at `/data`.
- **PID 1 signal handling (Node/Python):** These runtimes don't forward SIGTERM to children by default → slow/graceful shutdown broken. Add `--init` to `docker run`, or use `tini`/`dumb-init` as `ENTRYPOINT`, or ensure the app itself handles SIGTERM.
- **Private registry dependencies:** `RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci`. Never `COPY .npmrc` — the token ends up in a layer forever.

## Do not

- **`FROM ubuntu` / `FROM debian` for a single-binary app.** That's ~80 MB of attack surface you don't run.
- **`COPY . .` as the first COPY.** Destroys layer caching — every source edit reinstalls every dependency.
- **`USER root` in the final stage** just because a package install needed it. Do root work earlier, drop privileges last.
- **`apt-get upgrade` in the Dockerfile.** Pin a newer base image instead — upgrades in-place are non-reproducible.
- **`EXPOSE` as a security boundary.** It's documentation. The orchestrator decides what's actually published.
