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