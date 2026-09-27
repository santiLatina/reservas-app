# Etapa 1: Instalación de dependencias de producción
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# Etapa 2: Imagen final de ejecución
FROM node:20-alpine
WORKDIR /app

# Definición de variables de entorno y argumentos
ARG APP_VERSION=v1
ENV APP_VERSION=${APP_VERSION}
ENV NODE_ENV=production

# Copia de archivos asignando la propiedad al usuario no privilegiado 'node'
COPY --chown=node:node --from=deps /app/node_modules ./node_modules
COPY --chown=node:node . .

# Cambio al usuario no privilegiado
USER node

EXPOSE 3000
CMD ["node", "server.js"]