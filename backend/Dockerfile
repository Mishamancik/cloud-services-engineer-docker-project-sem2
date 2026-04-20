# Builder stage
FROM golang:1.17-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./

RUN go mod download

COPY . .

RUN go build -o main ./cmd/api


# Runtime stage
FROM alpine:latest

#TODO: Проверить необходимость установки ca-certificates
RUN addgroup -S momo -g 1000 \
 && adduser -S -G momo -u 1000 momo \
 && apk --no-cache add ca-certificates

WORKDIR /app

COPY --from=builder --chown=momo:momo /app/main .

EXPOSE 8081

USER momo

CMD ["./main"]
