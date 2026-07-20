# syntax=docker/dockerfile:1

##### Stage: base #####
FROM node:22-alpine AS base
WORKDIR /app
COPY package.json package-lock.json ./

##### Stage: dev #####
FROM base AS dev
ENV NODE_ENV=development
RUN npm install
COPY src ./src
CMD ["npm", "run", "dev"]

##### Stage: deps #####
FROM base AS deps
RUN npm ci --omit=dev

##### Stage: runtime#####
FROM node:22-alpine AS runtime
ENV NODE_ENV=production
WORKDIR /app

COPY --from=deps --chown=node:node /app/node_modules ./node_modules
COPY --chown=node:node package.json ./
COPY --chown=node:node src ./src

# hardening:
RUN rm -rf /usr/local/lib/node_modules/npm /usr/local/lib/node_modules/corepack /usr/local/bin/npm /usr/local/bin/npx /usr/local/bin/corepack

USER node

# Worker nema HTTP port i nema HEALTHCHECK na Docker/Compose razini:
CMD ["node", "src/worker.js"]
