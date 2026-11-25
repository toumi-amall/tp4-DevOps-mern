# TP4: Docker, Docker Compose and Jenkins (MERN)

**Course:** Introduction au DevOps, ISG Sousse, 3 LIG, 2026-2027

**Links**
- GitHub: https://github.com/toumi-amall/tp4-DevOps-mern
- Docker Hub (server): https://hub.docker.com/r/amaltoumi/mern-server
- Docker Hub (client): https://hub.docker.com/r/amaltoumi/mern-client

## Overview
MERN application (React client, Express API, MongoDB) containerized with Docker,
orchestrated with Docker Compose and deployed by a Jenkins pipeline.

Services:
- **mongodb**: image `mongo`, container port 27017, not published
- **server**: built from `./Server`, container port 9000, host port 9000
- **client**: built from `./Client`, container port 3000, host port 3001

## Prerequisites
Docker Desktop (with Docker Compose), Git, Jenkins, a Docker Hub account.

## 1. Clone the project
```bash
git clone https://github.com/wiem-kb/MyAppReact.git
```
Downloads the source code. It contains `client` (React, Vite) and `server` (Express).

## 2. Dockerfiles

### Server (`Server/Dockerfile`)
```dockerfile
FROM node:lts-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 9000
CMD ["npm", "start"]
```
- `FROM node:lts-alpine`: small Node.js base image.
- `WORKDIR /app`: working directory inside the container.
- `COPY package*.json ./`: copies only the dependency files first, so the install layer is cached.
- `RUN npm install`: installs the dependencies.
- `COPY . .`: copies the rest of the server code.
- `EXPOSE 9000`: documents the port used by the API.
- `CMD ["npm", "start"]`: starts the application.

`MONGO_URI` (the MongoDB connection URL) is not in the Dockerfile: it is given at run time.

### Client (`Client/Dockerfile`)
```dockerfile
FROM node:lts-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
RUN npm install -g serve
EXPOSE 3000
CMD ["serve", "-s", "dist", "-l", "3000"]
```
Same first steps as the server, then:
- `RUN npm run build`: builds the React app into static files.
- `RUN npm install -g serve`: installs a light HTTP server.
- `CMD ["serve", "-s", "dist", "-l", "3000"]`: serves the build on port 3000.

The project uses Vite, so the output folder is `dist` (not `build`).

### .dockerignore
Each folder has a `.dockerignore` file containing `node_modules`, so the local
dependencies are not copied into the image.

## 3. Build the images
```bash
docker build -t mern-server ./Server
docker build -t mern-client ./Client
```
`docker build` creates an image, `-t` gives it a name, and the last argument is the
folder containing the Dockerfile.

## 4. Manual run (without Compose)
```bash
docker network create mern-network
```
Creates a network so the containers can talk to each other by name.

```bash
docker run -d --name mongodb --network mern-network mongo
```
Starts MongoDB. `-d` runs it in the background, `--name` names the container,
`--network` attaches it to our network.

```bash
docker run -d --name server --network mern-network -p 9000:9000 -e MONGO_URI=mongodb://mongodb:27017/mydb mern-server
```
Starts the API. `-p 9000:9000` publishes the port (`host:container`).
`-e` sets the environment variable. The host name `mongodb` is the container name.

```bash
docker run -d --name client --network mern-network -p 3001:3000 mern-client
```
Starts the client. Port 3000 was already used by another project on my machine,
so the client is published on host port 3001 (the container still listens on 3000).

Useful commands:
```bash
docker ps                            # list running containers
docker logs server                   # show the logs of a container
docker rm -f client server mongodb   # remove the containers (before using Compose)
docker network rm mern-network       # remove the network
```

## 5. Docker Compose (`docker-compose.yml`)
```yaml
services:
  mongodb:
    image: mongo
    container_name: mongodb
    networks:
      - mern-network

  server:
    build: ./Server
    container_name: server
    ports:
      - "9000:9000"
    environment:
      - MONGO_URI=mongodb://mongodb:27017/mydb
    depends_on:
      - mongodb
    networks:
      - mern-network

  client:
    build: ./Client
    container_name: client
    ports:
      - "3001:3000"
    depends_on:
      - server
    networks:
      - mern-network

networks:
  mern-network:
    driver: bridge
```
- `services`: the three containers of the application.
- `build`: folder of the Dockerfile. `image`: ready-made image.
- `ports`: published ports (`host:container`).
- `environment`: sets `MONGO_URI` for the server.
- `depends_on`: start order (mongodb, then server, then client).
- `networks`: the `mern-network` bridge network, so services reach each other by service name.

```bash
docker compose up --build
```
Builds the images and starts all services.

```bash
docker compose down
```
Stops and removes the containers and the network.

Access:
- Client: `http://localhost:3001`
- API: `http://localhost:9000/api/health`

## 6. Jenkins pipeline (CI/CD)
Jenkins is installed on Windows and runs on `http://localhost:8081`.

### Configuration
1. Install the suggested plugins (Git, Pipeline).
2. **New Item**, name `tp4-mern-pipeline`, type **Pipeline**.
3. **Pipeline script from SCM**, SCM `Git`, repository
   `https://github.com/toumi-amall/tp4-DevOps-mern.git`, branch `*/main`.
4. Script Path: `Jenkinsfile`, then **Save** and **Build Now**.

### Jenkinsfile
```groovy
pipeline {
  agent any
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Build images') {
      steps { bat 'docker compose build' }
    }
    stage('Deploy') {
      steps {
        bat 'docker compose down || exit 0'
        bat 'docker compose up -d'
      }
    }
    stage('Test') {
      steps {
        bat 'ping -n 16 127.0.0.1 > nul'
        bat 'docker exec server wget -qO- http://localhost:9000/api/health'
      }
    }
  }
  post {
    failure { bat 'docker compose logs || exit 0' }
  }
}
```
Stages:
- **Checkout**: gets the code from GitHub.
- **Build images**: builds the Docker images with Compose.
- **Deploy**: removes the old containers, then starts the new ones in the background (`-d`).
- **Test**: waits about 15 seconds (`ping`), then calls the API health route inside the server container.

Other parts:
- `post { failure }`: prints the Compose logs if a stage fails.
- `bat`: runs Windows commands (`sh` does not exist on Windows).
- `|| exit 0`: ignores the error if there is nothing to stop.

![Pipeline stage view](screenshots/jenkins-stage-view.png)
![Console output](screenshots/jenkins-console.png)

## 7. Docker Hub
The images are published on Docker Hub:
- Server: https://hub.docker.com/r/amaltoumi/mern-server
- Client: https://hub.docker.com/r/amaltoumi/mern-client

```bash
docker login
```
Authenticates to Docker Hub.

```bash
docker tag mern-server amaltoumi/mern-server:latest
docker tag mern-client amaltoumi/mern-client:latest
```
Gives each local image the name `amaltoumi/image`, which Docker Hub requires.

```bash
docker push amaltoumi/mern-server:latest
docker push amaltoumi/mern-client:latest
```
Uploads the images to Docker Hub.

Anyone can now run the images with `docker pull amaltoumi/mern-server:latest`.

![Docker images](screenshots/docker-images.png)
![Docker Hub](screenshots/dockerhub.png)

## 8. Result
![Running containers](screenshots/docker-ps.png)
![Client](screenshots/client.png)
![API health](screenshots/api-health.png)

## 9. Problems encountered and solutions

**Container name already in use**
Solution: `docker rm -f <name>`, then run again.

**Multi-line command failed in PowerShell**
Solution: use one-line commands (backtick instead of `\`).

**Port 3000 already in use**
Solution: mapped the client to `3001:3000`.

**404 on the client**
Solution: `serve` pointed to `build`, but Vite outputs to `dist`.

**Jenkins: `Cannot run program "sh"`**
Solution: Jenkins runs on Windows, so `sh` was replaced with `bat`.

**Jenkins: `unknown command: docker compose`**
Solution: the service ran as SYSTEM, so it was switched to a local user account.

**Jenkins service error 1069**
Solution: the account had no real password (PIN only), so a password was set.