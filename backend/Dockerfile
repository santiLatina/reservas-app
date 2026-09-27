# Etapa 1: Dependencias
FROM    node:20-alpine AS builder
WORKDIR /reservas-api
COPY    package.json package-lock.json .
RUN     npm ci --omit=dev


# Etapa 2
FROM    node:20-alpine
WORKDIR /reservas-api
COPY    --from=builder /reservas-api/node_modules ./node_modules
COPY    db.js server.js .
ENV     NODE_ENV=production
ARG     APP_VERSION=v1
ENV     APP_VERSION=${APP_VERSION}
EXPOSE  3000
CMD     ["node", "db.js", "server.js"]