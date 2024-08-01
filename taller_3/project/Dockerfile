FROM node:14.17.4
WORKDIR /usr/src/app

COPY . .
RUN npm install express https

EXPOSE 3000
CMD ["node", "index.js"]