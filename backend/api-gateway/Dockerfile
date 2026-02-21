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

EXPOSE 3000
CMD ["node", "src/server.js"]
