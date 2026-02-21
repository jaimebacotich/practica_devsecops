# ================================
# Stage 1: deps
# Instala solo dependencias de producción
# ================================
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm install -g npm@latest && npm ci --omit=dev

# ================================
# Stage 2: runtime
# Imagen limpia con usuario no-root
# ================================
FROM node:22-alpine AS runtime

# Eliminar npm, npx y yarn del runtime (no se necesitan)
# Esto remueve vulnerabilidades de tar, glob y minimatch
RUN rm -rf /usr/local/lib/node_modules/npm \
    /usr/local/bin/npm /usr/local/bin/npx \
    /opt/yarn-* /usr/local/bin/yarn /usr/local/bin/yarnpkg \
    /usr/local/lib/node_modules/corepack

# Crear grupo y usuario sin privilegios
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copiar dependencias desde el stage anterior
COPY --from=deps /app/node_modules ./node_modules

# Copiar código fuente
COPY src/ ./src/

# Cambiar propietario y ejecutar como non-root
RUN chown -R appuser:appgroup /app
USER appuser

EXPOSE 3001
CMD ["node", "src/server.js"]