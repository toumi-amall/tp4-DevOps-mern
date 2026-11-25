# TP4: Docker and Docker Compose (MERN)

## Overview
MERN application containerized with Docker: a React (Vite) client, an Express API
and a MongoDB database, orchestrated with Docker Compose.

## Prerequisites
- Docker Desktop (with Docker Compose)

## 1. Dockerfiles
**Server** (`Server/Dockerfile`): node:lts-alpine base image, working directory `/app`,
copy `package*.json`, `npm install`, copy the sources, expose 9000, start with `npm start`.

**Client** (`Client/dockerfile`): node:lts-alpine base image, `npm install`,
`npm run build`, install `serve`, expose 3000, serve the `dist` folder.
Note: the project uses Vite, so the build output is `dist` and not `build`.

## 2. Manual run
```bash
docker build -t mern-server ./Server
docker build -t mern-client ./Client
docker network create mern-network
docker run -d --name mongodb --network mern-network mongo
docker run -d --name server --network mern-network -p 9000:9000 -e MONGO_URI=mongodb://mongodb:27017/mydb mern-server
docker run -d --name client --network mern-network -p 3001:3000 mern-client
```
Port 3000 was already used on my machine, so the client is published on host
port 3001 (the container still listens on 3000).

## 3. Docker Compose
```bash
docker compose up --build
```
- Client: http://localhost:3001
- API: http://localhost:9000/api/health

Services communicate through the service name (`mongodb`) as hostname.

## 4. Problems encountered
- Container name conflicts: fixed with `docker rm -f`.
- 404 on the client: `serve` pointed to `build`, but Vite outputs to `dist`.
- Port 3000 already in use: mapped to 3001.

## 5. Screenshots
(docker compose ps, the app in the browser, the API health route)