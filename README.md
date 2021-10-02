# Lecture Summary

## Section 8. Building a Multi-Container Application

- Create complex projects

## Section 9. "Dockerizing" Multiple Services

### - Dockerizing a React App

- Create a Dockerfile.dev in client folder

```Docker
FROM node:alpine

WORKDIR '/app'

COPY ./package.json ./
RUN npm install
COPY . .

CMD ["npm", "run", "start"]

```

- Run "docker build -f Dockerfile.dev ." in client folder
- Run "docker run -it _IMAGE_ID_" to make sure the server runs

### - Dockerizing Generic Node Apps

- Create a Dockerfile.dev in server and worker folders

```Docker
FROM node:14.14.0-alpine

WORKDIR '/app'

COPY ./package.json ./
RUN npm install
COPY . .

CMD ["npm", "run", "dev"]
```

- Run "docker build -f Dockerfile.dev" in both server and worker folders

### - Adding Postgres and redis as s Service

- Create a docker-compose-dev.yml in root folder

```yaml
version: '3'
services:
  postgres:
    image: 'postgres:latest'
    environment:
      - POSTGRES_PASSWORD=postgres_password
  redis:
    iamge: 'redis:latest'
```

### Docker-compose Config

- Edit docker-compose-dev.yml

```yaml
version: '3'
services:
  postgres:
    image: 'postgres:latest'
    environment:
      - POSTGRES_PASSWORD=postgres_password
  redis:
    image: 'redis:latest'
  api:
    build:
      dockerfile: Dockerfile.dev
      context: ./server
    volumes:
      - /app/node_modules
      - ./server:/app
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - PGUSER=postgres
      - PGHOST=postgres
      - PGDATABASE=postgres
      - PGPASSWORD=postgres_password
      - PGPORT=5432
```

- Run "docker-compose -f docker-compose-dev.yml up" to make sure all the services are up and running

### Add worker and client services

- Add below two services to docker-compose-dev.yml

```yaml
  client:
    stdin_open: true
    build:
      dockerfile: Dockerfile.dev
      context: ./client
    volumes:
      - /app/node_modules
      - ./client:/app
  worker:
    build:
      dockerfile: Dockerfile.dev
      context: ./worker
    volumes:
      - /app/node_modules
      - ./worker:/app
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
```

### Routing with Nginx

- Create a folder named "nginx"
- Create a file named "default.conf" in nginx folder

```default.conf
upstream client {
  server client:3000;
}

upstream api {
  server api:5000;
}

server {
  listen 80;

  location / {
    proxy_pass http://client;
  }

  location /api {
    rewrite /api/(.*) /$1 break;
    proxy_pass http://api;
  }
}
```

- Create Dockerfile.dev in nginx folder

```docker
FROM nginx

COPY ./default.conf /etc/nginx/conf.d/default.conf
```

- Add below to docker-compose-dev.yml

```yaml
  nginx:
    restart: always
    build:
      dockerfile: Dockerfile.dev
      context: ./nginx
    ports:
      - '3050:80'
    depends_on:
      - api
      - client
```

- Run "docker-compose -f docker-compose-dev.yml up --build" and hit localhost:3050

### Opening Websocket Connections

- Add below code in default.conf in nginx folder

```conf
location /sockjs-node {
  proxy_pass http://client;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "Upgrade"
}
```

- Build and run "docker-compose -f docker-compose-dev.yml up --build"

## Section 10. A Continuous Integration Workflow for Multiple Images

### Production Dockerfile

- Create Dockerfile in worker and server folder

```Dockerfile
FROM node:14.14.0-alpine

WORKDIR '/app'

COPY ./package.json ./
RUN npm install
COPY . .

CMD ["npm", "run", "start"]
```

- Create Dockerfile in nginx folder

```Dockerfile
FROM nginx
COPY ./default.conf /etc/nginx/conf.d/default.conf
```

### Altering Ngix's Listen Port

- Create a folder named nginx and add a file named default.conf in client folder

```yml
server {
  listen 3000;

  location / {
    root /usr/share/nginx/html;
    index index.html index.htm;
  }
}
```

- Create Dockerfile in client folder

```Dockerfile
FROM node:14.14.0-alpine as builder
WORKDIR '/app'
COPY ./package.json .
RUN npm install
COPY . .
RUN npm run build

FROM nginx
EXPOSE 3000
COPY ./nginx/default.conf etc/nginx/conf.d/default.conf
COPY --from=builder /app/build /usr/share/nginx/html
```

### Cleaning Up Tests

- Remove codes inside test function in client/src/App.test.js

### Github and Travis CI setup

