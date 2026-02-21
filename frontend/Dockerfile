# ================================
# Stage 1: build
# Compila la aplicación React/Vite
# ================================
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install -g npm@latest && npm install
COPY . .
RUN npm run build

# ================================
# Stage 2: serve
# Imagen nginx limpia con usuario no-root
# ================================
FROM nginx:alpine AS runtime

# nginx:alpine incluye usuario 'nginx' (uid=101) integrado
# Ajustar permisos para que nginx pueda correr sin root
RUN chown -R nginx:nginx /usr/share/nginx/html && \
    chown -R nginx:nginx /var/cache/nginx && \
    chown -R nginx:nginx /var/log/nginx && \
    touch /var/run/nginx.pid && \
    chown nginx:nginx /var/run/nginx.pid

COPY --from=build /app/dist /usr/share/nginx/html

USER nginx

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]