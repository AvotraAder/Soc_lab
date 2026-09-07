FROM node:20-alpine

RUN apk add --no-cache tini
RUN npm install -g n8n

EXPOSE 5678
ENTRYPOINT ["tini", "--"]
CMD ["n8n", "start"]
