# Base images come from the internal AITC ACR rather than Docker Hub.
ARG NODE_IMAGE=az01aitcdevacr.azurecr.io/baseimages/node:20-alpine
ARG NGINX_IMAGE=az01aitcdevacr.azurecr.io/baseimages/nginx:1.27-alpine

# ──────────────────────────────────────────────
# Stage 1: build the Vite bundle
# ──────────────────────────────────────────────
FROM ${NODE_IMAGE} AS build

# TE corporate proxy — build-time only, never baked into the runtime image.
ARG HTTP_PROXY
ARG HTTPS_PROXY
ARG NO_PROXY
ARG http_proxy
ARG https_proxy
ARG no_proxy

WORKDIR /app

# strict-ssl is disabled because the TE proxy terminates TLS, so npm cannot
# verify the intercepted certificate chain against its bundled roots.
RUN if [ -n "$HTTP_PROXY" ]; then \
        npm config set proxy "$HTTP_PROXY" && \
        npm config set https-proxy "$HTTPS_PROXY" && \
        npm config set strict-ssl false; \
    fi

# Install dependencies first so this layer caches independently of source edits.
COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build

# ──────────────────────────────────────────────
# Stage 2: serve the static build with nginx
# ──────────────────────────────────────────────
FROM ${NGINX_IMAGE} AS runtime

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80

# Use 127.0.0.1 rather than localhost: inside the container localhost resolves
# to ::1 first, but nginx listens on IPv4 only.
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://127.0.0.1:80/ || exit 1

CMD ["nginx", "-g", "daemon off;"]
