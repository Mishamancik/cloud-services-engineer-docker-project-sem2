# Builder stage
FROM node:16-alpine AS build

ARG VUE_APP_API_URL=/api

ENV VUE_APP_API_URL=$VUE_APP_API_URL

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

RUN npm run build


# Runtime stage
FROM nginx:alpine

RUN addgroup -S momo -g 1000 \
 && adduser -S -G momo -u 1000 momo \
 && mkdir -p /var/cache/nginx /var/run /var/log/nginx /run \
 && chown -R momo:momo /var/cache/nginx /var/run /var/log/nginx /usr/share/nginx/html /run \
 && rm -rf /usr/share/nginx/html/*

COPY --from=build --chown=momo:momo /app/dist /usr/share/nginx/html

COPY --chown=momo:momo nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

USER momo

CMD ["nginx", "-g", "daemon off;"]
