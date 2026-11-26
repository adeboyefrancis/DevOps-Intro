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

# ─── runtime stage ───
FROM gcr.io/distroless/static:nonroot
COPY --from=builder /quicknotes /quicknotes
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

# Output
{
  "User": "nonroot",
  "ExposedPorts": {
    "8080/tcp": {}
  },
  "Env": [
    "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    "SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt"
  ],
  "Entrypoint": [
    "/quicknotes"
  ],
  "WorkingDir": "/home/nonroot"
}
```
