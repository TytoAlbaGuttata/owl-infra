FROM node:20-alpine

RUN apk add --no-cache git

WORKDIR /app

RUN git clone https://github.com/stats-organization/github-stats-extended.git .

RUN npm install

EXPOSE 9000

CMD ["node", "express.js"]
