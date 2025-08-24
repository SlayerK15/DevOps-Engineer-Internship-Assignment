FROM node:18-alpine
WORKDIR /app
COPY package.json package-lock.json* ./
RUN if [ -f package-lock.json ]; then npm ci --omit=dev; else npm install --omit=dev; fi
COPY . .
# create entrypoint that sets MongoDB URL from env
RUN printf '%s\n' \
  '#!/bin/sh' \
  'echo "module.exports = { url: process.env.MONGO_URL || \"mongodb://mongo:27017/dd_db\" };" > app/config/db.config.js' \
  'exec node server.js' > /entrypoint.sh && chmod +x /entrypoint.sh

ENV NODE_ENV=production
EXPOSE 8080
CMD ["/entrypoint.sh"]
