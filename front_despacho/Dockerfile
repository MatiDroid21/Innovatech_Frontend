# ─── STAGE 1: Build ──────────────────────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci --silent

COPY . .

# Declarar ARGs para que Vite los embeba en el bundle durante el build
ARG VITE_API_URL
ARG VITE_VENTAS_URL
ARG VITE_DESPACHOS_URL

ENV VITE_API_URL=$VITE_API_URL
ENV VITE_VENTAS_URL=$VITE_VENTAS_URL
ENV VITE_DESPACHOS_URL=$VITE_DESPACHOS_URL

RUN npm run build

# ─── STAGE 2: Serve con Nginx ─────────────────────────────────
FROM nginx:alpine

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
