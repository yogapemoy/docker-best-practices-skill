# Docker 多服务部署最佳实践

> 生产级 Dockerfile 和 Docker Compose 架构指南 - Claude AI Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Docker](https://img.shields.io/badge/Docker-Official-blue)](https://www.docker.com/)
[![Claude AI](https://img.shields.io/badge/Claude_AI-Skill-purple)](https://claude.ai/skills)

## 概述

这是一个为 Claude AI 设计的技能包（Skill），提供 Docker 多服务部署的最佳实践指导。内容涵盖 Dockerfile 优化、Docker Compose 架构、健康检查、数据持久化和安全配置。

### 核心架构运行哲学

本 Skill 遵循以下 5 条必须遵守的原则：

1. **Dockerfile 不随环境变化** - Dockerfile 只定义"构建什么"，不定义"运行时配置"
2. **Compose 不包含真实 secret** - 所有敏感信息通过环境变量或 secret manager 注入
3. **所有环境差异 → `.env`** - 环境变量优先级：Shell > --env-file > .env > compose defaults
4. **容器可销毁，数据不可** - 容器是临时的，数据必须持久化到 volumes
5. **恢复能力 > 自动化炫技** - 优先考虑故障恢复和数据安全

### Dockerfile 构建五原则

1. **为合成而构建** - 把不变的放在易变的前面
2. **分离构建和运行** - 让最终的产物绝对纯粹
3. **最小权限运行** - 永远不要相信默认设置
4. **自动化健康检查** - 赋予系统治愈的能力
5. **凡事都要明确** - 清晰声明工作目录、端口和入口点

## 内容结构

```
docker-best-practices/
├── SKILL.md           # Claude AI 主技能文件
├── README.md          # 本文件
├── LICENSE            # MIT 许可证
├── references/        # 参考文档分类
│   ├── getting_started.md
│   ├── dockerfile.md
│   ├── compose.md
│   ├── security.md
│   └── other.md
├── assets/            # 示例项目和模板
└── scripts/           # 辅助脚本
```

## 安装方式

### 方法 1：通过 Claude AI 上传（推荐）

1. 下载最新版本的 `docker-best-practices.zip`
2. 访问 [Claude AI Skills](https://claude.ai/skills)
3. 点击 "Upload Skill"
4. 选择下载的 `.zip` 文件
5. 上传完成！

### 方法 2：手动安装

1. 克隆或下载本仓库
2. 将 `SKILL.md` 和 `references/` 目录打包为 `.zip` 文件
3. 按照方法 1 的步骤上传

## 使用指南

### 触发场景

这个 Skill 会在以下场景自动激活：

- 设计多服务 Docker 架构
- 编写或优化 Dockerfile
- 配置 Docker Compose
- 实现数据持久化策略
- 设置环境特定配置
- 实现容器健康检查
- 安全配置 Docker 容器

### 核心内容示例

#### 多阶段构建示例

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

#### Compose 环境变量管理

```yaml
# compose.yaml
services:
  web:
    image: "webapp:${TAG:-latest}"
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - DEBUG=${DEBUG:-false}

volumes:
  pgdata:
```

```bash
# .env
TAG=v1.5
DATABASE_URL=postgres://db:5432/app
DEBUG=false
```

#### 安全配置示例

```dockerfile
FROM python:3.12-slim

# 创建非 root 用户
RUN addgroup -g 1000 appgroup && \
    adduser -D -u 1000 -G appgroup appuser

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# 非特权运行
USER appuser
EXPOSE 8080/tcp
```

## 参考文档

Skill 包含以下分类参考文档：

| 文档 | 内容 | 页数 |
|------|------|------|
| `getting_started.md` | Dockerfile 最佳实践、多阶段构建、基础镜像选择 | 4 |
| `dockerfile.md` | 多阶段构建语法、BuildKit 差异、完整指令参考 | 2 |
| `compose.md` | 环境变量插值、.env 配置、生产部署模式 | 5 |
| `security.md` | 容器安全配置、权限管理、资源限制 | 1 |
| `other.md` | Docker run 命令、容器生命周期、网络基础 | 1 |

## 最佳实践速查

### Dockerfile 优化

- ✅ 使用多阶段构建减小镜像体积
- ✅ 选择官方 slim 或 alpine 镜像
- ✅ 合并 RUN 命令减少层数
- ✅ 使用 `.dockerignore` 排除不必要文件
- ✅ 以非 root 用户运行
- ✅ 添加健康检查
- ✅ 使用具体版本标签，避免 `latest`

### Compose 配置

- ✅ 使用 `.env` 文件管理环境变量
- ✅ 创建多环境配置文件（dev/staging/prod）
- ✅ 使用命名 volumes 持久化数据
- ✅ 不要在 compose.yaml 中存储真实 secret
- ✅ 设置资源限制（CPU、内存）

### 安全建议

- ✅ 使用最小权限原则运行容器
- ✅ 删除不必要的 Linux capabilities
- ✅ 设置资源限制防止资源耗尽
- ✅ 使用 secrets 管理敏感信息
- ✅ 定期更新基础镜像

## 数据来源

本 Skill 基于 Docker 官方文档生成：

- [Docker 最佳实践](https://docs.docker.com/develop/dev-best-practices/)
- [多阶段构建](https://docs.docker.com/build/building/multi-stage/)
- [Docker Compose 生产指南](https://docs.docker.com/compose/production/)
- [卷管理](https://docs.docker.com/storage/volumes/)
- [容器安全](https://docs.docker.com/engine/reference/run/#security-configuration)

## 贡献指南

欢迎提交 Issue 和 Pull Request！

### 更新 Skill

如需更新 Skill 内容：

1. 更新 `SKILL.md` 或相关参考文档
2. 重新打包为 `.zip` 文件
3. 上传到 Claude AI

### 添加新内容

1. 在 `references/` 目录添加新的 `.md` 文件
2. 在 `SKILL.md` 的 "Reference Files" 部分添加引用
3. 提交 PR

## 许可证

MIT License - 详见 [LICENSE](LICENSE) 文件

## 致谢

- Docker 官方文档团队
- Claude AI Skill 平台
- 所有贡献者

---

**Made with ❤️ for the Docker community**

如果这个 Skill 对你有帮助，请给一个 ⭐ Star！
