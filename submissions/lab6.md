# Lab 6: Containers: Dockerize Quicknotes app

### Task 1 - Multi-Stage Build

_Dockerfile_: Read only file that contains union mount file system of quicknotes which contains application, library, runtime and dependencies. Using a Multi build strategy for optimization by abstracting heavy lifting at the builder stage and cascading the final build in a distroless container. This ensure a light weight final image size with unwanted dependencies.

```ruby
# ─── builder stage ───
FROM golang:1.24 AS builder
WORKDIR /src
COPY go.mod ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags='-s -w' -o /quicknotes
RUN mkdir -p /data && chown -R 65532:65532 /data

# ─── runtime stage ───
FROM gcr.io/distroless/static:nonroot
COPY --from=builder --chown=65532:65532 /quicknotes /quicknotes
COPY --from=builder --chown=65532:65532 /data /data
COPY --chown=65532:65532 seed.json /seed.json
COPY seed.json /seed.json
EXPOSE 8080
USER nonroot
ENTRYPOINT ["/quicknotes"]
```

_Investigate final image_: Using `docker history <image>` to see the additional layers added to the base image and `docker inspect <image>`

```bash
docker images quicknotes:lab6

# Output
IMAGE             ID             DISK USAGE   CONTENT SIZE   EXTRA
quicknotes:lab6   919a790a19a8       15.1MB         3.17MB

docker inspect quicknotes:lab6 | jq '.[0].Config'
```

```json
// Output
{
  "User": "nonroot",
  "ExposedPorts": {
    "8080/tcp": {}
  },
  "Env": [
    "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    "SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt"
  ],
  "Entrypoint": ["/quicknotes"],
  "WorkingDir": "/home/nonroot"
}
```

### Task 2 - Compose + Health Check + Persistent volume

Test 1 - POST action _Starting docker compose_

```bash
docker compose up --build -d
curl -X POST -H 'Content-Type: application/json' -d '{"title":"durable","body":"survive a restart"}' http://localhost:8080/notes
curl -s http://localhost:8080/notes | grep durable
```

Test 2 - _Stopping docker compose without persistent data remove_

```bash
docker compose down
docker compose up -d && sleep 3
curl -s http://localhost:8080/notes | grep durable
```

```json
// Persistent Tests
// Output 1
{"id":1,"title":"durable","body":"survive a restart","created_at":"2026-11-28T17:29:15.887562979Z"}
// Output 2
[{"id":1,"title":"durable","body":"survive a restart","created_at":"2026-11-28T17:29:15.887562979Z"}]
// Output 3
[{"id":1,"title":"durable","body":"survive a restart","created_at":"2026-11-28T17:29:15.887562979Z"}]
```

### Trivy Scan Images

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:0.59.1 image --severity HIGH,CRITICAL --no-progress \
  quicknotes:lab6

# Output Summary
quicknotes:lab6 (debian 13.8)
=============================
Total: 0 (HIGH: 0, CRITICAL: 0)


quicknotes (gobinary)
=====================
Total: 23 (HIGH: 23, CRITICAL: 0)
```

### Design Question - Phase 1

## a) Layer Order

Docker caches layers top-to-bottom and invalidates everything after a changed layer. Copying `go.mod` and `go.sum` first and running `go mod download` before `COPY . .` keeps dependency downloads cached during code-only edits, speeding up rebuilds by avoiding unnecessary module re-downloads.

---

## b) `CGO_ENABLED=0` in `distroless/static`

Disables cgo to produce a static binary without external C library dependencies. Forgetting it in `distroless/static` can link against dynamic C libraries/loaders not present in the image, causing runtime startup failures.

---

## c) `gcr.io/distroless/static:nonroot`

- **In it:** Minimal runtime files, CA certificates, timezone data, and non-root user `65532` (`nonroot`).
- **Not in it:** No shell, package manager, or OS utilities (`curl`, `python3`, `ls`).
- **Why it matters for CVEs:** Minimal packages mean a smaller attack surface and fewer OS/library vulnerabilities.

---

## d) `-ldflags='-s -w'` and `-trimpath`

- **`-s`:** Strips the symbol table.
- **`-w`:** Strips DWARF debug info.
- **`-trimpath`:** Removes local build paths from the binary.
- **Cost:** Smaller binaries and no leaked local paths, but you lose debug symbols/DWARF info for debugging, profiling, and detailed stack traces.

### Design Question - Phase 2

## e) Healthchecking Distroless Without a Shell

**Chosen approach:** **Use a binary that is already in the image** (your own Go app binary supporting a health flag/subcommand, or relying on Docker's default process check if no health mode exists).

- **Why:** A binary already in the image avoids pulling extra images (`wget`/debug or sidecars), keeps the container self-contained, and requires no shell. (If your binary has no health subcommand, omit `healthcheck` and rely on Docker's default process-alive behavior rather than adding sidecar bloat for a simple lab app.)

---

## f) Named Volume Lifecycle (`quicknotes-data:/data`)

- **Why it survives `docker compose down`:** Named volumes are managed independently by Docker outside the container lifecycle. `docker compose down` stops and removes containers and networks, but leaves named volumes intact so data persists across restarts and rebuilds.
- **What destroys it:**
- `docker compose down -v` (or `--volumes`), which removes Compose-defined volumes.
- `docker volume rm quicknotes-data` (or `docker volume prune` when unused).

---

## g) `depends_on` Without `condition: service_healthy`

- **What it waits for:** Only that the dependency container has **started** (its main process is running), not that the app inside it is ready or passing health checks.
- **Bug it can cause:** Race conditions / startup errors—the dependent service starts and connects before the dependency is ready to accept traffic or finish migrations/initialization (e.g., connection refused or failed initial requests).
