# syntax=docker/dockerfile:1

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
CMD ["npm", "run", "dev"]

##### Stage: 
FROM base AS deps
RUN npm ci --omit=dev

##### Stage: runtime 
FROM node:22-alpine AS runtime
ENV NODE_ENV=production
WORKDIR /app

COPY --from=deps --chown=node:node /app/node_modules ./node_modules
COPY --chown=node:node package.json ./
COPY --chown=node:node src ./src

USER node

# Worker nema HTTP port i nema HEALTHCHECK na Docker/Compose razini:
# minimalna Alpine slika ne sadrzi procps (pgrep/ps s punim opcijama), a dodavanje
# tog paketa samo radi healthcheck-a bilo bi u suprotnosti s ciljem minimalne slike.
# Zdravlje workera u produkciji nadzire se kroz Kubernetes exec/liveness probe (CP5)
# i kroz observability/logove (worker ispisuje gresku i nastavlja petlju kod pada ovisnosti).

CMD ["node", "src/worker.js"]
