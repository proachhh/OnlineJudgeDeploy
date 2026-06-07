# Next-Generation Intelligent Online Judge — Deployment Guide

> Built with Python Django + Vue.js, integrating LLM, knowledge graphs, and deep learning recommendations.

---

## Environment Requirements

- **Linux (Ubuntu 22.04)** is recommended
- Windows users should use **WSL2** or a virtual machine (e.g., VMWare) to run a Linux subsystem; compatibility issues may arise otherwise
- Add your SSH public key to GitHub for cloning repositories

---

## I. Production Deployment

### 1. Download Images

Download 4 base images and 1 project image from the image server.

**Option A: Command Line (Recommended)**

```bash
wget http://47.108.158.48/judge.tar
wget http://47.108.158.48/neo4j.tar
wget http://47.108.158.48/postgres.tar
wget http://47.108.158.48/redis.tar
wget http://47.108.158.48/myoj_5.72.tar
```

> Always use the latest project image. Verify the version number.

**Option B: Browser Download**

Visit <http://47.108.158.48/> to download all images directly.

### 2. Load Images

```bash
docker load -i redis.tar
docker load -i postgres.tar
docker load -i judge.tar
docker load -i myoj_5.72.tar
docker load -i neo4j-5.26.13.tar
```

### 3. Clone Deployment Config

```bash
git clone git@github.com:proachhh/OnlineJudgeDeploy.git
cd OnlineJudgeDeploy
rm -f docker-compose.override.yml
```

### 4. Start All Containers

```bash
docker-compose up -d
```

> Depending on your Docker version, you may need `docker compose up -d`.

### Done

Access the frontend at `http://localhost`.

---

## II. Development Environment Setup

### 1. Download Images

Same as production:

```bash
wget http://47.108.158.48/judge.tar
wget http://47.108.158.48/neo4j.tar
wget http://47.108.158.48/postgres.tar
wget http://47.108.158.48/redis.tar
wget http://47.108.158.48/myoj_5.72.tar
```

### 2. Load Images

```bash
docker load -i redis.tar
docker load -i postgres.tar
docker load -i judge.tar
docker load -i myoj_5.72.tar
docker load -i neo4j-5.26.13.tar
```

### 3. Clone Source Code

Clone the backend and frontend repositories in a suitable parent directory:

```bash
# Backend
git clone git@github.com:proachhh/GraphOnlineJudge-backend.git

# Frontend
git clone git@github.com:proachhh/GraphOnlineJudge-frontend.git
```

### 4. Clone Deployment Config

```bash
git clone git@github.com:proachhh/OnlineJudgeDeploy.git
cd OnlineJudgeDeploy
```

### 5. Edit Local Mount Path

```bash
vim docker-compose.override.yml
```

Under `oj-backend:` → `volumes:`, replace the source mount path with your actual backend directory:

```yaml
volumes:
  - /home/your-username/GraphOnlineJudge-backend:/code
```

> Keep the `:/code` suffix and maintain correct indentation.

### 6. Start Containers

```bash
docker-compose up -d
```

### 7. Enter the Container and Initialize

```bash
docker exec -it onlinejudgedeploy-oj-backend-1 bash
```

> If `bash` is unavailable, use `sh`: `docker exec -it onlinejudgedeploy-oj-backend-1 sh`

Inside the container, run the following in order:

```bash
# Database migrations
python /code/manage.py makemigrations
python /code/manage.py migrate

# Build knowledge graph
python /code/manage.py build_knowledge_graph

# Set judge server token
python /code/manage.py shell -c "
from options.models import SysOptions
opt = SysOptions.objects.first()
if opt:
    opt.judge_server_token = 'my-dev-token-2026'
    opt.save()
    print('Token set successfully')
else:
    print('SysOptions not found')
"
```

### 8. Set Up Frontend Dev Environment

Navigate to the frontend directory and install Node.js 16 (nvm recommended):

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
source ~/.bashrc
nvm install 16
nvm use 16

# Install dependencies
npm install
```

### 9. Configure Frontend Proxy

Edit `vue.config.js` to proxy API requests to the backend container:

```js
module.exports = {
  devServer: {
    proxy: {
      '/api': {
        target: 'http://localhost:8001',
        changeOrigin: true
      }
    }
  }
}
```

### 10. Build DLL

```bash
npm run build:dll
```

### 11. Start the Frontend Dev Server

```bash
npm run dev
```

> Keep this terminal open — do not close it or press Ctrl+C. Consider using `screen`.

### 12. Start the Judge Worker (Optional)

Skip this step if judging functionality is not needed.

```bash
docker exec -it onlinejudgedeploy-oj-backend-1 python manage.py rundramatiq
```

### Development Environment Complete

Verify with the following URLs:

| Service | URL | Description |
|------|------|------|
| Frontend | `http://localhost` | Dev server with hot reload |
| Backend API | `http://localhost:8001/` | Append URL path, returns JSON |
| Graph Database | `http://localhost:7474` | Neo4j Browser, login with `neo4j` / `48691412kid` |

> Both frontend and backend support hot reload. Dev environment is memory-intensive; 4GB+ RAM recommended.

---

## Notes

1. Do **not** include `docker-compose.override.yml` in production — it mounts source code over the container image
2. Configure `JUDGE_SERVER_TOKEN` and database passwords via `.env`, not hardcoded in compose files
3. If server memory is tight, add `deploy.resources.limits.memory` to each service in `docker-compose.yml`
