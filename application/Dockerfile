FROM node:20-alpine

LABEL maintainer="your.email@example.com" \
      description="HRM Labs demo application"

WORKDIR /app

COPY package*.json ./

RUN npm ci --only=dev && \
    npm cache clean --force

COPY . .

RUN addgroup -S appgroup && \
    adduser -S appuser -G appgroup && \
    chown -R appuser:appgroup /app

USER appuser

EXPOSE 3000

ENV NODE_ENV=production \
    PORT=3000

HEALTHCHECK --interval=30s \
            --timeout=5s \
            --start-period=15s \
            --retries=3 \
  CMD node -e "require('http').get('http://127.0.0.1:3000/health', (r) => { process.exit(r.statusCode === 200 ? 0 : 1) }).on('error', () => process.exit(1))"

CMD ["node", "server.js"]