# 新一代智能在线判题系统 — 部署配置

> 基于 Python Django + Vue.js，深度融合大语言模型、知识图谱与深度学习推荐技术。

---

## 环境建议

- 本项目建议使用 **Linux（Ubuntu 22.04）** 运行
- Windows 用户建议使用 **WSL2** 运行 Linux 子系统，或使用虚拟机（如 VMWare）运行，否则可能出现兼容性问题
- 如需配置开发环境，建议向 Git 添加自己的公钥，以便克隆项目

---

## 一、生产环境部署

### 1. 下载镜像

从镜像服务器下载 4 个基础镜像与 1 个项目镜像。

**方式一：命令行下载（建议）**

```bash
wget http://47.108.158.48/judge.tar
wget http://47.108.158.48/neo4j.tar
wget http://47.108.158.48/postgres.tar
wget http://47.108.158.48/redis.tar
wget http://47.108.158.48/myoj_5.72.tar
```

> 项目镜像建议使用最新版，请确认版本号。

**方式二：浏览器直接下载**

直接访问 <http://47.108.158.48/> 即可下载所有镜像。

### 2. 加载镜像

```bash
docker load -i redis.tar
docker load -i postgres.tar
docker load -i judge.tar
docker load -i myoj_5.72.tar
docker load -i neo4j-5.26.13.tar
```

### 3. 克隆部署配置

```bash
git clone git@github.com:proachhh/OnlineJudgeDeploy.git
cd OnlineJudgeDeploy
rm -f docker-compose.override.yml
```

### 4. 启动所有容器

```bash
docker-compose up -d
```

> 因 Docker 版本不同，也可能是 `docker compose up -d`。

### 部署完成

通过 `http://localhost` 访问项目前端界面。

---

## 二、开发环境配置

### 1. 下载镜像

与生产环境相同：

```bash
wget http://47.108.158.48/judge.tar
wget http://47.108.158.48/neo4j.tar
wget http://47.108.158.48/postgres.tar
wget http://47.108.158.48/redis.tar
wget http://47.108.158.48/myoj_5.72.tar
```

### 2. 加载镜像

```bash
docker load -i redis.tar
docker load -i postgres.tar
docker load -i judge.tar
docker load -i myoj_5.72.tar
docker load -i neo4j-5.26.13.tar
```

### 3. 克隆前后端项目

在合适的父目录下分别克隆后端和前端：

```bash
# 后端
git clone git@github.com:proachhh/GraphOnlineJudge-backend.git

# 前端
git clone git@github.com:proachhh/GraphOnlineJudge-frontend.git
```

### 4. 克隆部署配置目录

```bash
git clone git@github.com:proachhh/OnlineJudgeDeploy.git
cd OnlineJudgeDeploy
```

### 5. 修改本地挂载路径

```bash
vim docker-compose.override.yml
```

找到 `oj-backend:` 下的 `volumes:`，将源码挂载路径替换为实际的后端目录：

```yaml
volumes:
  - /home/你的用户名/GraphOnlineJudge-backend:/code
```

> 保留 `:/code` 字样，注意缩进对齐。

### 6. 启动容器

```bash
docker-compose up -d
```

### 7. 进入容器执行初始化

```bash
docker exec -it onlinejudgedeploy-oj-backend-1 bash
```

> 如果报错用 `sh` 替代 `bash`：`docker exec -it onlinejudgedeploy-oj-backend-1 sh`

在容器内依次执行：

```bash
# 数据库迁移
python /code/manage.py makemigrations
python /code/manage.py migrate

# 构建知识图谱
python /code/manage.py build_knowledge_graph

# 设置判题服务器 Token
python /code/manage.py shell -c "
from options.models import SysOptions
opt = SysOptions.objects.first()
if opt:
    opt.judge_server_token = 'my-dev-token-2026'
    opt.save()
    print('Token 已设置')
else:
    print('未找到 SysOptions 对象')
"
```

### 8. 配置前端开发环境

进入前端目录，安装 Node.js 16（建议使用 nvm 管理版本）：

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
source ~/.bashrc
nvm install 16
nvm use 16

# 安装依赖
npm install
```

### 9. 配置前端代理

编辑 `vue.config.js`，让代理指向后端容器的映射端口：

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

### 10. 构建 DLL

```bash
npm run build:dll
```

### 11. 启动前端开发服务器

```bash
npm run dev
```

> 该终端不能关闭，不能 Ctrl+C。建议使用 `screen` 或新开终端。

### 12. 启动判题 Worker（可选）

如果不需要判题功能可以跳过。

```bash
docker exec -it onlinejudgedeploy-oj-backend-1 python manage.py rundramatiq
```

### 开发环境配置完毕

可通过以下地址验证：

| 服务 | 地址 | 说明 |
|------|------|------|
| 前端页面 | `http://localhost` | 前端热重载开发端口 |
| 后端 API | `http://localhost:8001/` | 后跟 URL 路径，返回 JSON |
| 图数据库 | `http://localhost:7474` | Neo4j Browser，账号 `neo4j` / `48691412kid` |

> 前后端均支持热重载，修改代码实时生效。开发环境占用内存较大，建议最低 4GB 以上。

---

## 注意事项

1. 生产环境 `docker-compose.yml` 中不要包含 `docker-compose.override.yml`，避免源码挂载覆盖镜像内代码
2. 生产环境的 `JUDGE_SERVER_TOKEN` 和数据库密码请在 `.env` 中配置，不要硬编码
3. 如果服务器内存紧张，可在 `docker-compose.yml` 中为各容器添加 `deploy.resources.limits.memory` 限制
