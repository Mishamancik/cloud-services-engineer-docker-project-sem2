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
