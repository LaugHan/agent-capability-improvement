# Tooling 提升：Docker Compose 完整模板

来源: VietnamTutor + Claude Code 泄露
日期: 2026-04-01

## 完整开发环境模板

```yaml
# docker-compose.yml
version: '3.9'

services:
  # ============ Database ============
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-postgres}
      POSTGRES_DB: ${POSTGRES_DB:-app}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # ============ Redis ============
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --appendfsync everysec
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3
    restart: unless-stopped

  # ============ Backend ============
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    volumes:
      - ./backend:/app
      - /app/node_modules
      - ./backend/.env:/app/.env:ro
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: development
    restart: unless-stopped

  # ============ Frontend ============
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    volumes:
      - ./frontend:/app
      - /app/node_modules
    depends_on:
      - backend
    ports:
      - "8080:8080"
    environment:
      VITE_API_URL: http://localhost:3000

  # ============ Worker (Celery) ============
  worker:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    command: celery -A worker worker --loglevel=info
    volumes:
      - ./backend:/app
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
    environment:
      - BROKER_URL=redis://redis:6379/0
      - RESULT_BACKEND=redis://redis:6379/1

  # ============ Object Storage (MinIO) ============
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER:-minioadmin}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD:-minioadmin}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  # ============ Email Testing (Mailhog) ============
  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "1025:1025"
      - "8025:8025"

  # ============ Database Admin (pgAdmin) ============
  pgadmin:
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: ${PGADMIN_EMAIL:-admin@admin.com}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_PASSWORD:-admin}
    ports:
      - "5050:80"
    depends_on:
      - postgres

# ============ Volumes ============
volumes:
  postgres_data:
  redis_data:
  minio_data:

# ============ Networks ============
networks:
  default:
    name: app-network
```

```sql
-- init/init.sql

-- Create extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Create tables
CREATE TABLE IF NOT EXISTS users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS orders (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id),
    total_cents INTEGER NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Seed data
INSERT INTO users (email, name) VALUES
    ('admin@example.com', 'Admin User'),
    ('test@example.com', 'Test User');
```

```makefile
# Makefile

.PHONY: up down logs build restart test shell db-reset db-shell clean

# Start services
up:
	docker-compose up -d

# Stop services
down:
	docker-compose down

# View logs
logs:
	docker-compose logs -f

# View specific service logs
logs-backend:
	docker-compose logs -f backend

# Restart a service
restart-backend:
	docker-compose restart backend

# Run tests
test:
	docker-compose exec backend npm test

# Shell into backend
shell-backend:
	docker-compose exec backend sh

# Shell into database
db-shell:
	docker-compose exec postgres psql -U postgres -d app

# Reset database (destroy volume)
db-reset:
	docker-compose down -v
	docker-compose up -d

# Rebuild without cache
build:
	docker-compose build --no-cache

# Clean up everything
clean:
	docker-compose down -v --remove-orphans
	docker network prune -f
```

```bash
# .env.example

# Database
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=app

# pgAdmin
PGADMIN_EMAIL=admin@admin.com
PGADMIN_PASSWORD=admin

# MinIO
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin

# App
NODE_ENV=development
```

## Docker Compose 最佳实践

### 1. 使用 Alpine 镜像
```yaml
image: postgres:16-alpine  # 比普通镜像小很多
```

### 2. Pin 版本
```yaml
image: nginx:1.25-alpine  # 而非 nginx:latest
```

### 3. Health Check 是必须的
```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U postgres"]
  interval: 5s
  timeout: 5s
  retries: 5
```

### 4. depends_on + condition
```yaml
depends_on:
  postgres:
    condition: service_healthy  # 而非简单的 depends_on
```

### 5. 安全的 Volume 挂载
```yaml
# Read-only 挂载
volumes:
  - ./config:/app/config:ro

# 排除 node_modules
volumes:
  - ./backend:/app
  - /app/node_modules  # 匿名 volume 覆盖
```

### 6. 环境变量分离
```
.env          # 本地开发
.env.prod     # 生产环境
.env.test     # 测试环境
```

### 7. 安全最佳实践
```yaml
# 不以 root 运行
user: "1000:1000"

# 只授予必要权限
cap_drop:
  - ALL
cap_add:
  - NET_BIND_SERVICE
```

### 8. YAML Anchors 复用
```yaml
x-service-default: &service-default
  restart: unless-stopped
  logging:
    driver: json-file
    options:
      max-size: "10m"
      max-file: "3"

services:
  backend:
    <<: *service-default
    # ... 其他配置
```

## 考试答题模板

当题目要求 Docker Compose 时，标准答案结构：

```
1. docker-compose.yml（核心配置）
   - version, services, volumes, networks
   - 每个 service 的 image/build, volumes, ports, depends_on, healthcheck
2. .env.example（环境变量模板）
3. init.sql（数据库初始化）
4. Makefile（常用命令）
```

## 参考

- https://vietnamtutor.com/docker-compose-best-practices-2026/
- https://help.apiyi.com/claude-code-source-leak-march-2026-impact-ai-agent-industry.html
