
##### Stage: base #####
FROM node:22-alpine AS base
WORKDIR /app
COPY package.json package-lock.json ./

##### Stage: dev #####
# Koristi se u compose.yaml
FROM base AS dev
ENV NODE_ENV=development
RUN npm install
COPY src ./src
EXPOSE 8080
CMD ["npm", "run", "dev"]

##### Stage: deps #####
FROM base AS deps
RUN npm ci --omit=dev

##### Stage: runtime (default target - produkcijska slika) #####
FROM node:22-alpine AS runtime
ENV NODE_ENV=production
WORKDIR /app

# Non-root korisnik 
COPY --from=deps --chown=node:node /app/node_modules ./node_modules
COPY --chown=node:node package.json ./
COPY --chown=node:node src ./src

# hardening: 
RUN rm -rf /usr/local/lib/node_modules/npm \
    /usr/local/lib/node_modules/corepack \
    /usr/local/bin/npm \
    /usr/local/bin/npx \
    /usr/local/bin/corepack

USER node
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD node -e "require('http').get('http://127.0.0.1:8080/healthz', r => process.exit(r.statusCode === 200 ? 0 : 1)).on('error', () => process.exit(1))"

CMD ["node", "src/server.js"]
