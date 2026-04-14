# Ubuntu 部署指南

本文提供一套适合 Ubuntu 服务器的生产部署方案，包含 Docker 部署、离线/内网部署、源码部署，以及 Nginx 与 systemd 的基础配置建议。

## 推荐方案

- **公网服务器**：优先使用 Docker 部署
- **内网/离线服务器**：使用 Docker Compose，同时自托管 draw.io
- **需要长期稳定运行**：配合 Nginx 和 `restart: unless-stopped` 或 systemd

---

## 方案一：Docker 部署（推荐）

### 1. 安装 Docker 与 Docker Compose

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
```

执行完成后重新登录终端，再用 `docker version` 和 `docker compose version` 确认安装成功。

### 2. 获取项目并准备环境变量

```bash
git clone https://github.com/DayuanJiang/next-ai-draw-io.git
cd next-ai-draw-io
cp env.example .env
```

编辑 `.env`，至少配置：

- `AI_MODEL`
- 对应提供商的 API Key，例如 `OPENAI_API_KEY`
- 如果配置了多个提供商，再显式设置 `AI_PROVIDER`

可参考 `/home/runner/work/next-ai-draw-io/next-ai-draw-io/docs/cn/ai-providers.md`。

### 3. 直接运行官方镜像

如果服务器可以访问 `embed.diagrams.net`，可以直接使用容器镜像：

```bash
docker run -d \
  --name next-ai-draw-io \
  --restart unless-stopped \
  -p 3000:3000 \
  --env-file .env \
  ghcr.io/dayuanjiang/next-ai-draw-io:latest
```

启动后访问：

- `http://服务器IP:3000`

### 4. 使用仓库内 Dockerfile 构建并运行

如果你希望自己构建镜像：

```bash
docker build -t next-ai-draw-io:local .
docker run -d \
  --name next-ai-draw-io \
  --restart unless-stopped \
  -p 3000:3000 \
  --env-file .env \
  next-ai-draw-io:local
```

### 5. 使用 Docker Compose

仓库根目录已经提供了 `docker-compose.yml`：

```bash
docker compose up -d --build
```

默认情况下：

- 应用端口：`3000`
- 自托管 draw.io 端口：`8080`

---

## 方案二：内网/离线部署

如果你的 Ubuntu 服务器无法访问 `embed.diagrams.net`，需要同时部署 draw.io。

### 1. 准备 `.env`

```bash
cp env.example .env
```

填写 AI 模型配置后，再修改 `docker-compose.yml` 中的构建参数。

仓库自带的 `/home/runner/work/next-ai-draw-io/next-ai-draw-io/docker-compose.yml` 已包含：

```yaml
services:
  drawio:
    image: jgraph/drawio:latest
    ports: ["8080:8080"]
  next-ai-draw-io:
    build:
      context: .
      args:
        - NEXT_PUBLIC_DRAWIO_BASE_URL=http://localhost:8080
    ports: ["3000:3000"]
    env_file: .env
    depends_on: [drawio]
```

### 2. 远程服务器必须改成浏览器可访问地址

如果用户是通过另一台电脑访问这台 Ubuntu 服务器，必须把：

```yaml
NEXT_PUBLIC_DRAWIO_BASE_URL=http://localhost:8080
```

改成类似：

```yaml
NEXT_PUBLIC_DRAWIO_BASE_URL=http://你的服务器IP:8080
```

或改成你的域名。

### 3. 启动服务

```bash
docker compose up -d --build
```

### 4. 非常重要的注意事项

- `NEXT_PUBLIC_DRAWIO_BASE_URL` 是**构建时变量**
- 改完后必须重新执行 `docker compose up -d --build`
- 这个值必须是**用户浏览器能直接访问**的地址
- **不要填写** `http://drawio:8080` 这类 Docker 内部服务名

如需更多背景说明，可参考 `/home/runner/work/next-ai-draw-io/next-ai-draw-io/docs/cn/offline-deployment.md`。

---

## 方案三：源码部署（不使用 Docker）

如果你更习惯在 Ubuntu 上直接运行 Node.js，也可以使用源码部署。

### 1. 安装 Node.js 24

仓库 Dockerfile 使用的是 Node 24，Ubuntu 上建议保持一致：

```bash
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
```

### 2. 安装依赖并配置环境变量

```bash
git clone https://github.com/DayuanJiang/next-ai-draw-io.git
cd next-ai-draw-io
npm install
cp env.example .env.local
```

编辑 `.env.local`，填入你的模型配置和 API Key。

### 3. 构建并启动生产服务

```bash
npm run build
npm run start
```

默认端口：

- 开发环境：`6002`（`npm run dev`）
- 生产环境：`6001`（`npm run start`）

---

## Nginx 反向代理建议

如果你要通过域名访问，建议在 Ubuntu 上使用 Nginx 反向代理。

### 1. 安装 Nginx

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl enable --now nginx
```

### 2. 反代应用主站

`/etc/nginx/sites-available/next-ai-draw-io.conf`

```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

如果你走源码部署，把 `3000` 改成 `6001`。

启用配置：

```bash
sudo ln -s /etc/nginx/sites-available/next-ai-draw-io.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### 3. 反代自托管 draw.io（可选）

如果你不想直接暴露 `8080`，可以给 draw.io 单独配置域名或路径。

例如单独域名：

```nginx
server {
    listen 80;
    server_name drawio.your-domain.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

然后把构建参数改成：

```bash
NEXT_PUBLIC_DRAWIO_BASE_URL=http://drawio.your-domain.com
```

---

## systemd / 守护进程建议

### Docker 部署

优先使用容器自动重启策略，例如：

```bash
docker run -d --restart unless-stopped ...
```

或使用 `docker compose` 并在服务中加入：

```yaml
restart: unless-stopped
```

### 源码部署

建议使用 systemd 托管 Node.js 进程。

`/etc/systemd/system/next-ai-draw-io.service`

```ini
[Unit]
Description=Next AI Draw.io
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/next-ai-draw-io
ExecStart=/usr/bin/npm run start
Restart=always
RestartSec=5
User=www-data
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

启用服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now next-ai-draw-io
sudo systemctl status next-ai-draw-io
```

---

## 子目录部署

如果应用不是部署在根路径，例如：

- `https://example.com/nextaidrawio`

则需要在构建前设置：

```bash
NEXT_PUBLIC_BASE_PATH=/nextaidrawio
```

注意：

- 这是 **构建时变量**
- 修改后必须重新构建镜像或重新执行 `npm run build`

---

## 部署完成后的检查项

- 应用首页可以正常打开
- 聊天请求可以正常返回
- draw.io 画布能够正常加载
- 上传图片或文本后功能正常
- 反向代理后静态资源路径正常
- 如果是离线部署，浏览器访问的 draw.io 地址与构建参数一致

如果你只是想快速上线，推荐优先使用：

- **公网**：Docker + 官方镜像 + Nginx
- **内网**：Docker Compose + 自托管 draw.io + Nginx
- **不用容器**：Node.js 24 + systemd + Nginx
