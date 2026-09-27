# Etapa 1: Build
FROM node:20-alpine AS build
WORKDIR /app

# Copiamos manifiestos e instalamos TODAS las dependencias
COPY package.json package-lock.json ./
RUN npm ci

# Copiamos el código fuente y generamos el build de produccion
COPY . .
RUN npm run build

# Etapa 2: Servidor web con Nginx
FROM nginxinc/nginx-unprivileged:1.27-alpine
COPY --from=build /app/dist /usr/share/nginx/html

# Variables de entorno
ARG APP_VERSION=v1
ENV APP_VERSION=${APP_VERSION}

# Variables requeridas para Nginx y envsubs
ENV API_HOST=reservas-api \
    API_PORT=3000 \
    NGINX_ENVSUBST_FILTER=API_

EXPOSE 8080

CMD ["nginx", "-g", "daemon off;"]