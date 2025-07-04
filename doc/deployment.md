# 部署指南

## 概述

NocoBase 支持多种部署方式，包括 Docker 部署、源码部署和云平台部署。本指南将详细介绍各种部署方案的配置和最佳实践。

## Docker 部署 (推荐)

### 使用 Docker Compose

#### 1. 创建 docker-compose.yml

```yaml
version: '3.8'

services:
  app:
    image: nocobase/nocobase:latest
    container_name: nocobase-app
    restart: unless-stopped
    ports:
      - "13000:80"
    environment:
      # 应用配置
      APP_ENV: production
      APP_PORT: 80
      APP_KEY: your-app-key-here
      
      # 数据库配置
      DB_DIALECT: postgres
      DB_HOST: postgres
      DB_PORT: 5432
      DB_DATABASE: nocobase
      DB_USER: nocobase
      DB_PASSWORD: nocobase_password
      
      # Redis 配置 (可选)
      REDIS_HOST: redis
      REDIS_PORT: 6379
      
      # 邮件配置 (可选)
      MAIL_DRIVER: smtp
      MAIL_HOST: smtp.gmail.com
      MAIL_PORT: 587
      MAIL_USERNAME: your-email@gmail.com
      MAIL_PASSWORD: your-email-password
      
    volumes:
      - nocobase_storage:/app/storage
      - nocobase_uploads:/app/storage/uploads
    depends_on:
      - postgres
      - redis
    networks:
      - nocobase

  postgres:
    image: postgres:15
    container_name: nocobase-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: nocobase
      POSTGRES_USER: nocobase
      POSTGRES_PASSWORD: nocobase_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - nocobase

  redis:
    image: redis:7-alpine
    container_name: nocobase-redis
    restart: unless-stopped
    volumes:
      - redis_data:/data
    networks:
      - nocobase

volumes:
  postgres_data:
  redis_data:
  nocobase_storage:
  nocobase_uploads:

networks:
  nocobase:
    driver: bridge
```

#### 2. 启动服务

```bash
# 创建并启动服务
docker-compose up -d

# 查看日志
docker-compose logs -f app

# 停止服务
docker-compose down

# 更新应用
docker-compose pull
docker-compose up -d
```

### 使用 Docker 单容器

```bash
# 使用 SQLite (适合测试)
docker run -d \
  --name nocobase \
  -p 13000:80 \
  -e APP_KEY=your-app-key \
  -e DB_DIALECT=sqlite \
  -e DB_STORAGE=/app/storage/db/nocobase.sqlite \
  -v nocobase-storage:/app/storage \
  nocobase/nocobase:latest

# 使用外部数据库
docker run -d \
  --name nocobase \
  -p 13000:80 \
  -e APP_KEY=your-app-key \
  -e DB_DIALECT=postgres \
  -e DB_HOST=your-db-host \
  -e DB_PORT=5432 \
  -e DB_DATABASE=nocobase \
  -e DB_USER=nocobase \
  -e DB_PASSWORD=your-password \
  -v nocobase-storage:/app/storage \
  nocobase/nocobase:latest
```

## 源码部署

### 环境要求

- Node.js >= 18
- Yarn 1.22.19
- 数据库 (PostgreSQL/MySQL/SQLite)
- Redis (可选，用于缓存和会话)

### 部署步骤

#### 1. 克隆代码

```bash
git clone https://github.com/nocobase/nocobase.git
cd nocobase
```

#### 2. 安装依赖

```bash
yarn install
yarn postinstall
```

#### 3. 配置环境变量

```bash
# 复制环境变量模板
cp .env.example .env

# 编辑配置文件
vim .env
```

#### 4. 环境变量配置

```bash
# .env 文件内容
APP_ENV=production
APP_KEY=your-secret-app-key
APP_PORT=13000

# 数据库配置
DB_DIALECT=postgres
DB_HOST=localhost
DB_PORT=5432
DB_DATABASE=nocobase
DB_USER=nocobase
DB_PASSWORD=your-password
DB_LOGGING=false

# Redis 配置
REDIS_HOST=localhost
REDIS_PORT=6379

# 邮件配置
MAIL_DRIVER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=your-email@gmail.com
MAIL_PASSWORD=your-password

# 文件存储
STORAGE_TYPE=local
STORAGE_BASE_URL=http://localhost:13000

# JWT 配置
JWT_SECRET=your-jwt-secret
```

#### 5. 构建和启动

```bash
# 构建应用
yarn build

# 安装应用 (首次部署)
yarn nocobase install

# 启动应用
yarn start

# 或使用 PM2 管理进程
npm install -g pm2
pm2 start ecosystem.config.js
```

### PM2 配置

```javascript
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: 'nocobase',
      script: './packages/app/server/lib/index.js',
      cwd: './',
      instances: 1,
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
        APP_ENV: 'production',
      },
      error_file: './logs/err.log',
      out_file: './logs/out.log',
      log_file: './logs/combined.log',
      time: true,
    },
  ],
};
```

## Nginx 反向代理

### 基础配置

```nginx
server {
    listen 80;
    server_name your-domain.com;
    
    # 重定向到 HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;
    
    # SSL 证书配置
    ssl_certificate /path/to/your/cert.pem;
    ssl_certificate_key /path/to/your/key.pem;
    
    # SSL 安全配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    # 客户端最大请求体大小
    client_max_body_size 100M;
    
    # 代理到 NocoBase
    location / {
        proxy_pass http://127.0.0.1:13000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
    
    # WebSocket 支持
    location /ws {
        proxy_pass http://127.0.0.1:13000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # 静态文件缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        proxy_pass http://127.0.0.1:13000;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
    
    # 安全头
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
}
```

### 负载均衡配置

```nginx
upstream nocobase_backend {
    server 127.0.0.1:13000;
    server 127.0.0.1:13001;
    server 127.0.0.1:13002;
    
    # 健康检查
    keepalive 32;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;
    
    # SSL 配置...
    
    location / {
        proxy_pass http://nocobase_backend;
        # 其他代理配置...
    }
}
```

## 数据库配置

### PostgreSQL

```bash
# 安装 PostgreSQL
sudo apt update
sudo apt install postgresql postgresql-contrib

# 创建数据库和用户
sudo -u postgres psql
CREATE DATABASE nocobase;
CREATE USER nocobase WITH PASSWORD 'your-password';
GRANT ALL PRIVILEGES ON DATABASE nocobase TO nocobase;
\q

# 配置连接
# 编辑 /etc/postgresql/15/main/postgresql.conf
listen_addresses = 'localhost'

# 编辑 /etc/postgresql/15/main/pg_hba.conf
local   nocobase        nocobase                                md5
host    nocobase        nocobase        127.0.0.1/32            md5

# 重启服务
sudo systemctl restart postgresql
```

### MySQL

```bash
# 安装 MySQL
sudo apt update
sudo apt install mysql-server

# 安全配置
sudo mysql_secure_installation

# 创建数据库和用户
sudo mysql
CREATE DATABASE nocobase CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'nocobase'@'localhost' IDENTIFIED BY 'your-password';
GRANT ALL PRIVILEGES ON nocobase.* TO 'nocobase'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## 监控和日志

### 系统监控

```bash
# 安装监控工具
npm install -g pm2

# 启动应用监控
pm2 start ecosystem.config.js
pm2 monit

# 查看日志
pm2 logs nocobase

# 重启应用
pm2 restart nocobase

# 保存 PM2 配置
pm2 save
pm2 startup
```

### 日志配置

```javascript
// 在应用配置中设置日志
{
  logger: {
    level: 'info',
    format: 'json',
    transports: [
      {
        type: 'console',
      },
      {
        type: 'file',
        filename: 'logs/app.log',
        maxSize: '10m',
        maxFiles: 5,
      },
      {
        type: 'file',
        level: 'error',
        filename: 'logs/error.log',
        maxSize: '10m',
        maxFiles: 5,
      },
    ],
  },
}
```

## 备份和恢复

### 数据库备份

```bash
# PostgreSQL 备份
pg_dump -h localhost -U nocobase -d nocobase > backup_$(date +%Y%m%d_%H%M%S).sql

# PostgreSQL 恢复
psql -h localhost -U nocobase -d nocobase < backup_20231201_120000.sql

# MySQL 备份
mysqldump -u nocobase -p nocobase > backup_$(date +%Y%m%d_%H%M%S).sql

# MySQL 恢复
mysql -u nocobase -p nocobase < backup_20231201_120000.sql
```

### 文件备份

```bash
# 备份上传文件
tar -czf uploads_backup_$(date +%Y%m%d_%H%M%S).tar.gz storage/uploads/

# 备份完整存储目录
tar -czf storage_backup_$(date +%Y%m%d_%H%M%S).tar.gz storage/
```

### 自动备份脚本

```bash
#!/bin/bash
# backup.sh

BACKUP_DIR="/path/to/backups"
DATE=$(date +%Y%m%d_%H%M%S)

# 创建备份目录
mkdir -p $BACKUP_DIR

# 数据库备份
pg_dump -h localhost -U nocobase -d nocobase > $BACKUP_DIR/db_backup_$DATE.sql

# 文件备份
tar -czf $BACKUP_DIR/storage_backup_$DATE.tar.gz storage/

# 清理旧备份 (保留7天)
find $BACKUP_DIR -name "*.sql" -mtime +7 -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Backup completed: $DATE"
```

```bash
# 添加到 crontab
crontab -e

# 每天凌晨2点执行备份
0 2 * * * /path/to/backup.sh >> /var/log/nocobase_backup.log 2>&1
```

## 性能优化

### 应用优化

```javascript
// 生产环境配置
{
  // 启用集群模式
  cluster: {
    enable: true,
    workers: require('os').cpus().length,
  },
  
  // 缓存配置
  cache: {
    store: 'redis',
    host: 'localhost',
    port: 6379,
    ttl: 3600,
  },
  
  // 数据库连接池
  database: {
    pool: {
      max: 20,
      min: 5,
      acquire: 30000,
      idle: 10000,
    },
  },
}
```

### 数据库优化

```sql
-- PostgreSQL 优化
-- 调整配置参数
shared_buffers = 256MB
effective_cache_size = 1GB
work_mem = 4MB
maintenance_work_mem = 64MB

-- 创建索引
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_posts_created_at ON posts(created_at);
CREATE INDEX idx_posts_status ON posts(status);
```

## 安全配置

### 防火墙设置

```bash
# UFW 防火墙配置
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow 80
sudo ufw allow 443
sudo ufw deny 13000  # 禁止直接访问应用端口
```

### 应用安全

```javascript
// 安全配置
{
  // CORS 配置
  cors: {
    origin: ['https://your-domain.com'],
    credentials: true,
  },
  
  // 安全头
  helmet: {
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        scriptSrc: ["'self'"],
        imgSrc: ["'self'", "data:", "https:"],
      },
    },
  },
  
  // 速率限制
  rateLimit: {
    windowMs: 15 * 60 * 1000, // 15分钟
    max: 100, // 最多100个请求
  },
}
```

---

*本部署指南涵盖了 NocoBase 的各种部署方案和最佳实践，帮助您在生产环境中稳定运行 NocoBase 应用。*
