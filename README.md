# Пельменная momo

Проект контейнеризации приложения с использованием Docker и Docker Compose.

![CI](https://github.com/Mishamancik/cloud-services-engineer-docker-project-sem2/actions/workflows/deploy.yaml/badge.svg)

## Архитектура проекта
Система состоит из двух сервисов:
- Backend - Go API
- Frontend - Vue.js и Nginx

Контейнеры запускаются через Docker Compose и взаимодействуют через внутренние сети Docker.

## Запуск проекта
Предварительная сборка контейнеров
```
docker compose build
```
Простой запуск
```
docker compose up -d
```
Проверка состояния
```
docker compose ps
```
Остановка
```
docker compose down
```

## Доступ к приложению
| Сервис | Адрес | Приложение |
| :--- | :--- | :--- |
| Backend | http://localhost:8080/api/orders | Postman |
| Frontend | http://localhost:8080 | Браузер |

## Dockerfile
Основные особенности:
- Реализованы multi-stage builds
- Оптимизированы размеры образов (минимизация слоев, отсутствие лишних инструментов)
- Используются непривилегированные пользователи
- Используются легковесные базовые образы (alpine и alpine-slim)
- Лишние файлы при сборке исключаются .dockerignore
- Выстроен эффективный порядок инструкций для кеширования
- Настроены healthchecks

### Backend
Обоснование шагов:
- ARG: базовый образ определен в качестве аргумента. Можно быстро выполнить сборку на другом образе, чтобы проверить совместимость с новой версией go или сравнить размер итоговых образов
- FROM: базовый образ builder stage подставлен в виде аргумента
- WORKDIR: определена рабочая директория
- COPY: отдельно копируются зависимости (они меняются реже исходного кода)
- RUN: устанавливаются зависимости (этот шаг обычно закеширован, так как зависимоти меняются редко)
- COPY: отдельно копируется исзодный код (в нем часто бывают изменения)
- RUN: сборка бинарного файла приложения (тоже закеширована, но при изменении исходного кода идет заново)
- FROM: для runtime stage используется alpine-образ (в distroless нет инструментов для healthcheck)
- WORKDIR: определена рабочая директория
- RUN: создается пользователь momo, становится владельцем рабочей директории
- COPY: копирование бинарного файла из builder stage с передачей владения пользователю momo
- EXPOSE: открывается правильный порт 8081
- USER: определяется рабочий пользователь для контейнера
- CMD: команда по умолчанию при запуске контейнера (запускает приложение и может быть переопределена)
- HEALTHCHECK: проверка отклика приложения каждые 30 секунд. Первая проверка проводится чуть раньше: через 10 секунд, так как от этого сервиса будет зависеть запуск fronted, и 30 секунд для запуска слишком долго.

<details closed>
<summary>Полный текст Dockerfile для backend</summary>

```
# Default values for arguments
ARG GOLANG_DOCKER_IMAGE_VERSION=golang:1.17-alpine


# Builder stage
FROM ${GOLANG_DOCKER_IMAGE_VERSION} AS builder

WORKDIR /app

COPY go.mod go.sum ./

RUN go mod download

COPY . .

RUN go build -o main ./cmd/api


# Runtime stage
FROM alpine:latest

WORKDIR /app

RUN addgroup -S momo -g 1000 \
 && adduser -S -G momo -u 1000 momo \
 && chown momo:momo /app

COPY --from=builder --chown=momo:momo /app/main .

EXPOSE 8081

USER momo

CMD ["./main"]

HEALTHCHECK --start-period=10s --interval=30s --timeout=3s --retries=2 \
  CMD wget -O- http://localhost:8081/health
```

</details>

### Frontend
Обоснование шагов:
- ARG: базовые образы для build и runtime stage определены в качестве аргументов. Также в качестве аргумента определена ссылка для api. Это глобальные аргументы, указанные в начале файла до секции FROM. (Такой подход позволяет указывать значение один раз)
- FROM: базовый образ builder stage подставлен в виде аргумента
- ARG: чтобы использовать глобальный аргумент внутри секции FROM, нужно объявить его
- ENV: значение переменной окружения внутри контейнера подставляется из аргумента
- WORKDIR: определена рабочая директория
- COPY: отдельно копируются зависимости (они меняются реже исходного кода)
- RUN: устанавливаются зависимости (этот шаг обычно закеширован, так как зависимоти меняются редко)
- COPY: отдельно копируется исзодный код (в нем часто бывают изменения)
- RUN: сборка файлов для fontend-приложения (тоже закеширована, но при изменении исходного кода идет заново)
- FROM: для runtime stage используется alpine-образ nginx, версия подставляется из аргумента
- RUN: создается пользователь momo, ему даются права на рабочие директории nginx (предварительно мягко обеспечивается их начличие через mkdir -p). Из /usr/share/nginx/html/ удаляется первоначальное содержание. Пакетный менеджер устанавливает curl, флаг --no-cache снижает размер образа.
- COPY: копирование frontend-файлов из builder stage с передачей владения пользователю momo
- COPY: копирование конфигурации nginx из проекта с передачей владения пользователю momo
- EXPOSE: открывается правильный порт 80
- USER: определяется рабочий пользователь для контейнера
- CMD: команда по умолчанию при запуске контейнера (запускает приложение и может быть переопределена)
- HEALTHCHECK: проверка отклика приложения каждые 30 секунд

<details closed>
<summary>Полный текст Dockerfile для frontend</summary>

```
# Default values for arguments
ARG NODE_DOCKER_IMAGE_VERSION=node:16-alpine
ARG NGINX_DOCKER_IMAGE_VERSION=nginx:alpine-slim
ARG VUE_APP_API_URL=/api


# Builder stage
FROM ${NODE_DOCKER_IMAGE_VERSION} AS build

ARG VUE_APP_API_URL # Global ARG has to be declared inside stage to be available after FROM

ENV VUE_APP_API_URL=${VUE_APP_API_URL}

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

RUN npm run build


# Runtime stage
FROM ${NGINX_DOCKER_IMAGE_VERSION}

RUN addgroup -S momo -g 1000 \
 && adduser -S -G momo -u 1000 momo \
 && mkdir -p /var/cache/nginx /var/run /var/log/nginx /usr/share/nginx/html /run \
 && chown -R momo:momo /var/cache/nginx /var/run /var/log/nginx /usr/share/nginx/html /run \
 && rm -rf /usr/share/nginx/html/* \
 && apk --no-cache add curl

COPY --from=build --chown=momo:momo /app/dist /usr/share/nginx/html

COPY --chown=momo:momo nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

USER momo

CMD ["nginx", "-g", "daemon off;"]

HEALTHCHECK --interval=30s --timeout=3s --retries=2 \
  CMD curl -f http://localhost/momo-store/
```

</details>

### Размеры образов
```
IMAGE                 ID             DISK USAGE   CONTENT SIZE   EXTRA
momo-backend:1.0.1    a09fffabc1a6       24.7MB             0B   U
momo-frontend:1.0.2   f2f18582879d       19.9MB             0B   U
```

### Конфигурируемость
Чтобы конфигурировать backend-образ, достаточно отредактировать параметры в команде ниже. В ней можно изменить версию базового образа go и тег сборки.
```
docker build \
--build-arg GOLANG_DOCKER_IMAGE_VERSION="golang:1.17-alpine" \
-t "backend:1.0.1" ./backend/
```

Чтобы конфигурировать frontend-образ, достаточно отредактировать параметры в команде ниже. В ней можно изменить версию базового образа node и nginx, адрес api и тег сборки.
```
docker build \
  --build-arg NODE_DOCKER_IMAGE_VERSION="node:16-alpine" \
  --build-arg NGINX_DOCKER_IMAGE_VERSION="nginx:alpine-slim" \
  --build-arg VUE_APP_API_URL="/api" \
  -t frontend:1.0.2 \
  ./frontend/
```

## Compose
Основные особенности:
- Настроена зависимость frontend от backend service healthy (ждем backend, чтобы присылать ему API-запросы)
- Используются healthchecks из Dockerfile
- Настроены изолированные сети для сервисов и общая сеть для коммуникации
- Настроены volumes для примера
- Настроены профили для dev и prod
- Реализовано горизонтальное масштабирование (балансировка запросов к backend через nginx во frontend)
- Настроены ограничения ресурсов, политики перезапуска, права доступа, capabilities, security scanning, приведен пример docker secrets, read-only fs.

<details closed>
<summary>Полный текст docker-compose.yml</summary>

```
services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
      args:
        GOLANG_DOCKER_IMAGE_VERSION: ${GOLANG_DOCKER_IMAGE_VERSION:-golang:1.26-alpine}
    image: momo-backend:${MOMO_BACKEND_VERSION:-recent}
    restart: unless-stopped
    expose:
      - "8081"
    networks:
      - main_network
      - backend
    read_only: true
    volumes:
      - backend:/example_volume
    cpus: 0.5
    mem_limit: 512m
    mem_reservation: 256m
    memswap_limit: 512m # Когда memswap_limit == mem_limit, использование swap запрещено
    pids_limit: 200
    cap_drop: 
      - ALL
    security_opt:
      - no-new-privileges:true
    secrets:
      - db_connection

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      args:
        VUE_APP_API_URL: ${VUE_APP_API_URL:-/api}
        NGINX_DOCKER_IMAGE_VERSION: ${NGINX_DOCKER_IMAGE_VERSION:-nginx:alpine-slim}
        NODE_DOCKER_IMAGE_VERSION: ${NODE_DOCKER_IMAGE_VERSION:-node:16-alpine}
    image: momo-frontend:${MOMO_FRONTEND_VERSION:-recent}
    depends_on:
      backend:
        condition: service_healthy
    restart: unless-stopped
    ports:
      - "8080:80"
    networks:
      - main_network
      - frontend
    read_only: true
    tmpfs:
      - /var/cache/nginx:uid=1000,gid=1000,mode=0755 # нужно явно указать пользователя momo
      - /var/run:uid=1000,gid=1000,mode=0755
      - /run:uid=1000,gid=1000,mode=0755
      - /tmp:uid=1000,gid=1000,mode=1777
    volumes:
      - frontend:/example_volume
    cpus: 0.5
    mem_limit: 512m
    mem_reservation: 256m
    memswap_limit: 512m
    pids_limit: 200
    cap_drop: 
      - ALL
    security_opt:
      - no-new-privileges:true

networks:
  main_network:
    driver: bridge
  frontend:
    driver: bridge
    internal: true
  backend:
    driver: bridge
    internal: true

volumes:
  backend:
  frontend:

secrets:
  db_connection:
    file: ./db_connection_secret
```

</details>

### Конфигурируемость
Для конфигурации приложения скопируйте содержние .env.example в .env рядом с compose-файлом и измените необходимые параметры.
```
# Backend
MOMO_BACKEND_VERSION="4.0.1"
GOLANG_DOCKER_IMAGE_VERSION="golang:1.17-alpine"

# Frontend
MOMO_FRONTEND_VERSION="4.0.2"
NODE_DOCKER_IMAGE_VERSION="node:16-alpine"
NGINX_DOCKER_IMAGE_VERSION="nginx:alpine-slim"
VUE_APP_API_URL="/api"
```

### Горизонтальное масштабирование
Для горизотального масштабирования подходит только stateless backend, так как для frontend просто собираются статичные файлы. Для runtime во frontend используется nginx, который работает как точка входа и обеспечивает балансировку запросов к множественным backend-инстансам. Backend-сервис не имеет закрепленного порта на хосте (это нужно для корретного масштабирования). 

Чтобы масштабировать backend, используйте соответствующий флаг в команде запуска:
``` 
docker compose up -d --scale backend=3 
```

### Профили для dev и prod
Для запуска приложения в dev/prod окружении, используйте профиль при помощи команды ниже. Профили перезаписывают существующие поля из базового compose-файла, либо добавляют новые.
```
 docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

<details closed>
<summary>docker-compose.dev.yml</summary>

```
services:
  backend:
    image: momo-backend:${MOMO_BACKEND_VERSION:-recent}-dev
    environment:
      - NODE_ENV=development
    networks:
      - main_network_dev
      - backend_dev
    volumes:
      - backend_dev:/app

  frontend:
    image: momo-frontend:${MOMO_FRONTEND_VERSION:-recent}-dev
    environment:
      - NODE_ENV=development
    networks:
      - main_network_dev
      - frontend_dev
    volumes:
      - frontend_dev:/usr/share

networks:
  main_network_dev:
    driver: bridge
  frontend_dev:
    driver: bridge
    internal: true
  backend_dev:
    driver: bridge
    internal: true

volumes:
  backend_dev:
  frontend_dev:
```

</details>

<details closed>
<summary>docker-compose.prod.yml</summary>

```
services:
  backend:
    image: momo-backend:${MOMO_BACKEND_VERSION:-recent}-prod
    environment:
      - NODE_ENV=production
    restart: always
    networks:
      - main_network_prod
      - backend_prod
    volumes:
      - backend_prod:/app

  frontend:
    image: momo-frontend:${MOMO_FRONTEND_VERSION:-recent}-prod
    environment:
      - NODE_ENV=production
    restart: always
    networks:
      - main_network_prod
      - frontend_prod
    volumes:
      - frontend_prod:/usr/share

networks:
  main_network_prod:
    driver: bridge
  frontend_prod:
    driver: bridge
    internal: true
  backend_prod:
    driver: bridge
    internal: true

volumes:
  backend_prod:
  frontend_prod:
```

</details>

### Сканирование образов
В CI/CD добавлен шаг сканирования образов при помощи trivy. Обнаруженные уязвимости проявляются во вкладке "Security and quality" в формате SARIF. Чтобы проверки выполнялись автоматически по расписанию, можно сделать отдельный workflow с запуском по cron (при помощи секции on cron). Ниже приведены результаты сканирования и фрагмент workflow.

Backend
```
Report Summary

┌────────────────────────────────────┬──────────┬─────────────────┬─────────┐
│               Target               │   Type   │ Vulnerabilities │ Secrets │
├────────────────────────────────────┼──────────┼─────────────────┼─────────┤
│ momo-backend:1.0.1 (alpine 3.23.4) │  alpine  │        0        │    -    │
├────────────────────────────────────┼──────────┼─────────────────┼─────────┤
│ app/main                           │ gobinary │        3        │    -    │
└────────────────────────────────────┴──────────┴─────────────────┴─────────┘
Legend:
- '-': Not scanned
- '0': Clean (no security findings detected)


app/main (gobinary)
===================
Total: 3 (UNKNOWN: 0, LOW: 0, MEDIUM: 3, HIGH: 0, CRITICAL: 0)
```

Frontend
```
Report Summary

┌─────────────────────────────────────┬────────┬─────────────────┬─────────┐
│               Target                │  Type  │ Vulnerabilities │ Secrets │
├─────────────────────────────────────┼────────┼─────────────────┼─────────┤
│ momo-frontend:1.0.2 (alpine 3.23.4) │ alpine │        0        │    -    │
└─────────────────────────────────────┴────────┴─────────────────┴─────────┘
Legend:
- '-': Not scanned
- '0': Clean (no security findings detected)
```

</details>

<details closed>
<summary>Фрагмент workflow с trivy</summary>

```
  run-trivy-image-scan:
    needs: build_and_push_to_docker_hub
    permissions:
      contents: read # for actions/checkout to fetch code
      security-events: write # for github/codeql-action/upload-sarif to upload SARIF results
      actions: read # only required for a private repository by github/codeql-action/upload-sarif to get the Action run status
    name: Run Trivy Image Scan
    runs-on: ubuntu-latest
    steps:
      - name: Run backend vulnerability scan
        uses: aquasecurity/trivy-action@7b7aa264d83dc58691451798b4d117d53d21edfe
        with:
          image-ref: 'docker.io/${{ secrets.DOCKER_USER }}/docker-project-backend:latest'
          format: 'template'
          template: '@/contrib/sarif.tpl'
          output: 'trivy-backend.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload backend scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-backend.sarif'
          category: backend

      - name: Run frontend vulnerability scan
        uses: aquasecurity/trivy-action@7b7aa264d83dc58691451798b4d117d53d21edfe
        with:
          image-ref: 'docker.io/${{ secrets.DOCKER_USER }}/docker-project-frontend:latest'
          format: 'template'
          template: '@/contrib/sarif.tpl'
          output: 'trivy-frontend.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload frontend scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-frontend.sarif'
          category: frontend
```

</details>

### Проверка docker-bench-security
Некоторые результаты с WARN-статусом, которые были устранены после security-hardening
```
[WARN]       * Container running with root FS mounted R/W: momo-backend-1
[WARN]       * Container running with root FS mounted R/W: momo-frontend-1
[WARN]       * PIDs limit not set: momo-backend-1
[WARN]       * PIDs limit not set: momo-frontend-1
[WARN]       * Privileges not restricted: momo-backend-1
[WARN]       * Privileges not restricted: momo-frontend-1
[WARN]      * No SecurityOptions Found: momo-backend-1
[WARN]      * No SecurityOptions Found: momo-frontend-1
```

### Security hardening
В официальном образе nginx privilleged-порты доступны всем пользователям (а не только root). Поэтому capability NET_BIND_SERVICE контейнеру с пользователем momo не требуется. В итоге у обоих контейнеров можно забрать все capabilities.

Контейнеру с go можно включить read-only fs. Для read-only nginx потребовалось создать tmpfs в тех местах, где контейнеру нужно писать данные, и явно указать владельца momo с правами. 

Для backend настроено монтирование Docker Secret в `/run/secrets/db_connection`.

Контейнерам установлены лимиты по cpu, memory, swap, PIDs, 
