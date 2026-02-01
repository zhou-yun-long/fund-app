# 部署到腾讯云服务器

## 前置条件

- 腾讯云服务器（CentOS/Ubuntu）
- 已安装 Node.js 18+
- 已安装 Nginx
- 域名（可选）

## 1. 本地构建

```bash
# 安装依赖
npm install

# 构建生产版本
npm run build
```

构建完成后会生成 `dist` 目录。

## 2. 上传文件到服务器

### 方式一：SCP 上传

```bash
# 压缩 dist 目录
tar -czvf dist.tar.gz dist

# 上传到服务器
scp dist.tar.gz root@你的服务器IP:/var/www/

# SSH 登录服务器
ssh root@你的服务器IP

# 解压
cd /var/www
tar -xzvf dist.tar.gz
mv dist fund-app
```

### 方式二：Git 拉取（推荐）

```bash
# 服务器上安装 Node.js（如未安装）
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# 克隆仓库
cd /var/www
git clone 你的仓库地址 fund-app
cd fund-app

# 安装依赖并构建
npm install
npm run build
```

## 3. 配置 Nginx

创建 Nginx 配置文件：

```bash
sudo vim /etc/nginx/conf.d/fund-app.conf
```

写入以下内容：

```nginx
server {
    listen 80;
    server_name 你的域名或IP;

    root /var/www/fund-app/dist;
    index index.html;

    # Gzip 压缩
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;

    # 静态资源缓存
    location /assets {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # SPA 路由支持
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API 代理（如需要）
    # location /api {
    #     proxy_pass https://fundgz.1234567.com.cn;
    # }
}
```

测试并重载 Nginx：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 4. 配置 HTTPS（推荐）

使用 Certbot 申请免费 SSL 证书：

```bash
# 安装 Certbot
sudo apt install certbot python3-certbot-nginx

# 申请证书（自动配置 Nginx）
sudo certbot --nginx -d 你的域名

# 自动续期
sudo certbot renew --dry-run
```

## 5. 防火墙配置

```bash
# 开放 80 和 443 端口
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
```

同时在腾讯云控制台的安全组中放行 80/443 端口。

## 6. 自动部署脚本（可选）

创建 `deploy.sh`：

```bash
#!/bin/bash
cd /var/www/fund-app
git pull origin main
npm install
npm run build
echo "部署完成: $(date)"
```

添加执行权限：

```bash
chmod +x deploy.sh
```

## 常见问题

### 页面刷新 404

确保 Nginx 配置了 `try_files $uri $uri/ /index.html`。

### 跨域问题

天天基金 API 使用 JSONP，不存在跨域问题。如果使用其他 API，需要配置 Nginx 代理。

### 基金列表加载失败

确保 `fund-list.json` 文件存在于 `dist` 目录中。
