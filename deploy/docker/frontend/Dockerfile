FROM node:20-alpine AS builder

WORKDIR /build

COPY ruoyi-ui-vue3/package*.json /build/
RUN npm ci

COPY ruoyi-ui-vue3 /build
RUN npm run build:prod

FROM nginx:1.27-alpine

COPY deploy/docker/frontend/nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /build/dist /usr/share/nginx/html

EXPOSE 80
