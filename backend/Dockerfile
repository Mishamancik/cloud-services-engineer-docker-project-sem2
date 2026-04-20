# Builder stage
FROM golang:1.17-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./

RUN go mod download

COPY . .

RUN go build -o main ./cmd/api


# Runtime stage
FROM alpine:latest

RUN addgroup -S momo -g 1000 \
 && adduser -S -G momo -u 1000 momo

WORKDIR /app

COPY --from=builder --chown=momo:momo /app/main .

EXPOSE 8081

USER momo

CMD ["./main"]

HEALTHCHECK --interval=30s --timeout=3s --retries=2 \
  CMD wget -O- http://localhost:8081/health
