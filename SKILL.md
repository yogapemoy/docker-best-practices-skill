---
name: docker-best-practices
description: Docker 多服务部署最佳实践 - 遵循生产级 Dockerfile 和 Docker Compose 架构原则。核心原则：Dockerfile 不随环境变化、compose 不包含真实 secret、所有环境差异通过 .env 管理、容器可销毁但数据不可、恢复能力优于自动化炫技。涵盖：Dockerfile 优化、多服务 compose 架构、健康检查、数据持久化、安全配置。
---

# Docker-Best-Practices Skill

Docker 多服务部署最佳实践 - 生产级 Dockerfile 和 Docker Compose 架构指南。

## When to Use This Skill

This skill should be triggered when:
- **Designing multi-service Docker architectures** for production deployment
- **Writing or optimizing Dockerfiles** for size, security, and performance
- **Configuring Docker Compose** for multi-container applications
- **Implementing data persistence** strategies with volumes and mounts
- **Setting up environment-specific configurations** using .env files
- **Implementing container health checks** and monitoring
- **Securing Docker containers** with proper user permissions and capabilities
- **Optimizing container resource usage** (CPU, memory, I/O limits)
- **Deploying applications** from development to production with Docker
- **Troubleshooting Docker deployment issues** related to networking, storage, or configuration

## 核心架构运行哲学

**必须遵守的 5 条原则：**

1. **Dockerfile 不随环境变化**
   - Dockerfile 只定义"构建什么"，不定义"运行时配置"
   - 所有环境差异（dev/staging/prod）通过 `.env` 文件注入
   - 同一个 Dockerfile 应该能在所有环境使用

2. **Compose 不包含真实 secret**
   - `compose.yaml` 只包含 secret 的占位符或引用
   - 真实密钥从 `.env`、secret manager 或挂载文件读取
   - 永远不要将 `compose.yaml` 包含的 secret 提交到版本控制

3. **所有环境差异 → `.env`**
   - 环境变量优先级：Shell > --env-file > .env > compose defaults
   - 每个环境有独立的 `.env.{environment}` 文件
   - `.env.example` 提供模板，但不包含真实值

4. **容器可销毁，数据不可**
   - 容器是 ephemeral（临时的），随时可以删除重建
   - 数据必须持久化到 volumes 或外部存储
   - 状态与容器分离，重建不影响数据

5. **恢复能力 > 自动化炫技**
   - 优先考虑故障恢复和数据安全
   - 避免过度复杂的自动化脚本
   - 简单的备份/恢复流程胜过复杂的编排

## Dockerfile 构建五原则

**必须遵循的 5 条构建原则：**

1. **为合成而构建，把不变的放在易变的前面**
   ```dockerfile
   # 不变的底层依赖（很少变化）
   FROM python:3.12-slim AS base
   WORKDIR /app

   # 依赖安装（偶尔变化）
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt

   # 应用代码（经常变化）
   COPY . .
   ```

2. **分离构建和运行，让最终的产物绝对纯粹**
   ```dockerfile
   # 构建阶段：包含所有构建工具
   FROM node:20 AS builder
   WORKDIR /app
   COPY package*.json ./
   RUN npm ci
   COPY . .
   RUN npm run build

   # 运行阶段：只包含运行时必需
   FROM node:20-slim
   COPY --from=builder /app/dist ./dist
   COPY --from=builder /app/node_modules ./node_modules
   USER node
   CMD ["node", "dist/server.js"]
   ```

3. **最小权限运行，永远不要相信默认设置**
   ```dockerfile
   FROM alpine:3.19
   # 创建非 root 用户
   RUN addgroup -g 1000 appgroup && \
       adduser -D -u 1000 -G appgroup appuser
   # 只开放必需的端口
   EXPOSE 8080
   # 以非 root 用户运行
   USER appuser
   # 删除不必要的 capabilities
   # 运行时：--cap-drop=ALL --cap-add=NET_BIND_SERVICE
   ```

4. **自动化健康检查，赋予系统治愈的能力**
   ```dockerfile
   HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
     CMD curl -f http://localhost:8080/health || exit 1

   # 或使用自定义脚本
   HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
     CMD /app/scripts/healthcheck.sh
   ```

5. **凡事都要明确，清晰的声明工作目录、暴露的端口和入口点**
   ```dockerfile
   # 明确的基础镜像版本
   FROM python:3.12-slim

   # 明确的工作目录
   WORKDIR /app

   # 明确的环境变量
   ENV PYTHONUNBUFFERED=1 \
       PYTHONDONTWRITEBYTECODE=1 \
       PATH="/app/bin:${PATH}"

   # 明确的暴露端口
   EXPOSE 8080/tcp

   # 明确的卷挂载点
   VOLUME ["/app/data", "/app/logs"]

   # 明确的入口点
   ENTRYPOINT ["python", "-m", "myapp"]
   CMD ["--config", "/app/config/config.yaml"]
   ```

## Quick Reference

### Dockerfile Best Practices

**1. Use multi-stage builds for size reduction**
```dockerfile
# syntax=docker/dockerfile:1
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-slim
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]
```
*Source: Official Docker documentation - Multi-stage builds*

**2. Choose minimal, trusted base images**
```dockerfile
# Good: Official slim images
FROM python:3.12-slim

# Better: Alpine-based (even smaller)
FROM python:3.12-alpine

# Best: distroless for production (minimal, auto-updates)
FROM gcr.io/distroless/python3-debian12
```
*Source: Official Docker best practices*

**3. Combine RUN commands to reduce layers**
```dockerfile
# Bad: Multiple layers
RUN apt-get update
RUN apt-get install -y python3
RUN apt-get clean

# Good: Single layer with cleanup
RUN apt-get update && \
    apt-get install -y --no-install-recommends python3 && \
    rm -rf /var/lib/apt/lists/*
```
*Source: Official Docker best practices*

**4. Use .dockerignore to exclude unnecessary files**
```
# .dockerignore
node_modules
npm-debug.log
.git
.env.local
*.md
tests/
```
*Source: Official Docker documentation*

**5. Run as non-root user**
```dockerfile
RUN adduser --disabled-password --gecos '' appuser
USER appuser
```
*Source: Official Docker security documentation*

**6. Implement health checks**
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```
*Source: Official Docker documentation*

**7. Use specific versions, not `latest`**
```dockerfile
# Bad: Unpredictable updates
FROM ubuntu:latest

# Good: Reproducible builds
FROM ubuntu:24.04
```
*Source: Official Docker best practices*

### Docker Compose Patterns

**8. Environment variable interpolation**
```yaml
# compose.yaml
services:
  web:
    image: "webapp:${TAG:-latest}"
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - DEBUG=${DEBUG:-false}
```

```bash
# .env
TAG=v1.5
DATABASE_URL=postgres://db:5432/app
```
*Source: Official Docker Compose documentation*

**9. Multi-environment configuration**
```yaml
# compose.base.yaml (common configuration)
services:
  web:
    build: .
    ports:
      - "8000:8000"

# compose.production.yaml (production overrides)
services:
  web:
    restart: always
    environment:
      - NODE_ENV=production
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
```

```bash
# Development
docker compose up

# Production
docker compose -f compose.base.yaml -f compose.production.yaml up -d
```
*Source: Official Docker Compose production guide*

**10. Volume persistence with named volumes**
```yaml
services:
  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=${DB_PASSWORD}

volumes:
  pgdata:
    # Named volume persists across container restarts
```
*Source: Official Docker volumes documentation*

**11. Using secrets in Compose**
```yaml
services:
  web:
    image: myapp:latest
    secrets:
      - db_password
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```
*Source: Official Docker security documentation*

### Data Persistence Patterns

**12. Volume for persistent data**
```bash
# Create and use a named volume
docker volume create mydata
docker run -d --name app \
  --mount source=mydata,target=/app/data \
  myapp:latest
```
*Source: Official Docker volumes documentation*

**13. Bind mount for development**
```bash
# Sync local files into container
docker run -d --name dev \
  --mount type=bind,source=$(pwd)/src,target=/app/src \
  myapp:latest
```
*Source: Official Docker documentation*

**14. Read-only volume for shared data**
```yaml
services:
  web:
    image: nginx:alpine
    volumes:
      - static_content:/usr/share/nginx/html:ro

volumes:
  static_content:
```
*Source: Official Docker volumes documentation*

### Container Security

**15. Drop unnecessary capabilities**
```bash
docker run -d --name secure_app \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  myapp:latest
```
*Source: Official Docker security documentation*

**16. Run with resource limits**
```bash
docker run -d --name app \
  --memory="512m" \
  --memory-reservation="256m" \
  --cpus="0.5" \
  --pids-limit 100 \
  myapp:latest
```
*Source: Official Docker run documentation*

**17. Run as non-root**
```bash
docker run -d --name app \
  --user 1000:1000 \
  myapp:latest
```
*Source: Official Docker security documentation*

### Production Deployment

**18. Health check in docker run**
```bash
docker run -d --name app \
  --health-cmd="curl -f http://localhost/health || exit 1" \
  --health-interval=30s \
  --health-timeout=3s \
  --health-retries=3 \
  myapp:latest
```
*Source: Official Docker run documentation*

**19. Update and redeploy a service**
```bash
# Rebuild and recreate without affecting dependencies
docker compose up -d --build web
```
*Source: Official Docker Compose production guide*

**20. Backup and restore volumes**
```bash
# Backup a volume
docker run --rm --volumes-from app_container \
  -v $(pwd):/backup \
  alpine tar czf /backup/backup.tar.gz /data

# Restore to new container
docker run --rm --volumes-from new_app_container \
  -v $(pwd):/backup \
  alpine tar xzf /backup/backup.tar.gz
```
*Source: Official Docker volumes documentation*

## Reference Files

This skill includes comprehensive documentation in `references/`:

### Core Documentation

- **getting_started.md** (4 pages, confidence: medium)
  - Dockerfile best practices
  - Multi-stage builds
  - Base image selection
  - Environment variable management
  - Rebuilding and caching strategies
  - `.dockerignore` usage

- **dockerfile.md** (2 pages, confidence: medium)
  - Multi-stage build syntax
  - Build stage naming and targeting
  - External image usage
  - BuildKit vs legacy builder differences
  - Complete Dockerfile instruction reference

- **compose.md** (5 pages, confidence: medium)
  - Environment variable interpolation
  - .env file configuration
  - Variable precedence rules
  - Volume management in Compose
  - Production deployment patterns

- **security.md** (1 page, confidence: medium)
  - Container security configuration
  - Linux capabilities management
  - User permissions
  - Resource constraints
  - Health check parameters

- **other.md** (1 page, confidence: medium)
  - Docker run command reference
  - Container lifecycle management
  - Networking basics
  - Filesystem mounts

### Usage Guidance

Use these reference files when you need:
- **Detailed syntax**: Check the appropriate reference file for complete syntax
- **Examples**: Each reference contains real-world examples from official docs
- **Troubleshooting**: Reference files contain common issues and solutions
- **Best practices**: Official recommendations for production deployments

## Working with This Skill

### For Beginners

1. **Start with the concepts**: Understand the core principles before diving into syntax
2. **Use the quick reference**: The examples above cover the most common patterns
3. **Check reference files**: When you need detailed information about specific commands

### For Intermediate Users

1. **Study multi-stage builds**: Essential for production-grade images
2. **Learn Compose patterns**: Critical for multi-service applications
3. **Understand volumes**: Data persistence is key to production deployments
4. **Implement security**: Never run production containers without security hardening

### For Advanced Users

1. **Optimize build caching**: Structure Dockerfiles to maximize cache hits
2. **Resource management**: Set appropriate CPU, memory, and I/O limits
3. **Multi-environment configs**: Use Compose overrides for dev/staging/prod
4. **Health monitoring**: Implement comprehensive health checks and logging
5. **Secrets management**: Use proper secret handling, never commit credentials

### Common Workflows

**Building a production image:**
1. Start with a minimal base image
2. Use multi-stage builds to reduce size
3. Combine RUN commands to minimize layers
4. Add health checks
5. Run as non-root user
6. Test locally before pushing

**Deploying with Compose:**
1. Create base `compose.yaml` with service definitions
2. Use `.env` for environment-specific values
3. Create override files (e.g., `compose.production.yaml`) for different environments
4. Use named volumes for persistent data
5. Never commit secrets to version control

**Managing data:**
1. Use named volumes for persistent data
2. Use bind mounts for development (hot reload)
3. Implement backup strategies for volumes
4. Document volume contents and purpose

## Notes

- **Source**: Official Docker documentation (docs.docker.com)
- **Coverage**: Dockerfiles, Docker Compose, volumes, security, and deployment
- **Best practices**: All examples follow official Docker recommendations
- **Multi-environment**: Patterns for dev, staging, and production
- **Security**: Emphasis on secure configurations and secrets management

## Key Takeaways

1. **Separate concerns**: Dockerfiles (build), Compose (orchestration), .env (configuration)
2. **Multi-stage builds**: Use them to create small, secure production images
3. **Never commit secrets**: Use environment variables and secret management
4. **Containers are ephemeral**: Design with data persistence in mind
5. **Security by default**: Minimal images, non-root users, dropped capabilities
6. **Reproducibility**: Pin image versions, use specific tags, document dependencies
7. **Health checks**: Always implement health checks for production services
8. **Resource limits**: Set CPU, memory, and I/O limits to prevent runaway containers

## Updating

To refresh this skill with updated documentation:
1. Re-run the scraper with the same configuration
2. The skill will be rebuilt with the latest information from official Docker documentation
