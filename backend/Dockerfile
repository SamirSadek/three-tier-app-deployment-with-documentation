FROM node:18-alpine

WORKDIR /usr/src/app

# Install dependencies
COPY package.json ./
RUN npm install --production

# Copy source
COPY . .

ENV NODE_ENV=production

EXPOSE 3000

CMD ["node", "src/server.js"]
