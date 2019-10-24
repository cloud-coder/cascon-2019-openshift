FROM node:10

# Create app directory
WORKDIR /usr/src/app

RUN mkdir -p /usr/src/app

COPY * /usr/src/app

RUN npm install

COPY . /usr/src/app

EXPOSE 8080
CMD [ "node", "server.js" ]