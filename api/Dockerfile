FROM node:24-alpine AS base
WORKDIR /app
COPY package*.json ./

FROM base AS dev
RUN npm install
COPY src ./src
USER node
CMD ["npm", "run", "dev"]

FROM base AS production
RUN npm ci --omit=dev
RUN apk upgrade --no-cache && \
    rm -rf /usr/local/lib/node_modules/npm \
           /usr/local/lib/node_modules/corepack \
           /usr/local/bin/npm /usr/local/bin/npx /usr/local/bin/corepack
COPY src ./src
USER node
EXPOSE 8080
CMD ["node", "src/server.js"]