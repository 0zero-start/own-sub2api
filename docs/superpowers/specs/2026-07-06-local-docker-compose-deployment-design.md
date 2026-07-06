# Sub2API 本地 Docker Compose 部署设计

## 目标

在 Windows Docker Desktop 上运行官方 Sub2API，并将完整官方源码历史同步到
`0zero-start/own-sub2api`。运行时账号、API 密钥、数据库、日志和部署密钥只保存在本机。

## 仓库关系

- `origin` 指向 `https://github.com/0zero-start/own-sub2api.git`，用于推送当前项目。
- `upstream` 指向 `https://github.com/Wei-Shaw/sub2api.git`，用于获取官方更新。
- 保留官方 `main` 分支的完整历史；本地修改通过正常提交推送到 `origin`。
- 拉取新的 Docker 镜像不会自动修改源码或 GitHub 仓库。官方源码更新需要从
  `upstream/main` 获取、合并并推送。

## 部署架构

使用官方 `deploy/docker-compose.local.yml` 和官方预构建镜像：

- `sub2api`：Sub2API 应用容器。
- `postgres`：持久化账号、API Key、配置和业务数据。
- `redis`：缓存及队列数据。
- `sub2api-network`：三个服务之间的专用 Docker 网络。

应用仅绑定宿主机 `127.0.0.1:8080`，不向局域网公开。PostgreSQL 和 Redis
不映射宿主机端口。

## 数据与密钥隔离

本地运行数据使用绑定目录：

- `deploy/data/`
- `deploy/postgres_data/`
- `deploy/redis_data/`
- `deploy/.env`

上述路径均由项目 `.gitignore` 排除。`.env` 包含自动生成的 PostgreSQL 密码、
JWT 密钥和 TOTP 加密密钥；管理员密码由 Sub2API 首次启动时自动生成并从容器日志读取。
推送前必须通过 `git status --ignored`、`git check-ignore` 和暂存区敏感文件检查确认这些内容
未进入提交。

## 启动流程

1. 从 `deploy/.env.example` 生成本机 `deploy/.env`。
2. 使用密码学安全随机数生成 PostgreSQL、JWT 和 TOTP 密钥。
3. 设置 `BIND_HOST=127.0.0.1`、`SERVER_PORT=8080`、`TZ=Asia/Shanghai`。
4. 创建三个本地持久化目录。
5. 校验 Compose 配置后拉取镜像并后台启动三个服务。
6. 从首次启动日志读取自动生成的管理员密码，只向本机用户显示，不写入 Git。

## 故障处理

- 若镜像拉取失败，保留配置和数据目录，报告 Docker 网络错误后重试。
- 若容器未健康，检查 `docker compose ps` 和应用、PostgreSQL、Redis 日志，不推送完成状态。
- 若 `8080` 端口冲突，停止部署并报告占用进程，不擅自终止其他程序。
- 若 GitHub 尚未授权，完成本地部署和提交后，使用 GitHub CLI 浏览器授权再推送。
- 不执行删除数据目录、强制推送或覆盖远程历史的操作。

## 验收标准

- `docker compose config --quiet` 成功。
- `sub2api`、`postgres`、`redis` 三个容器均为运行且健康状态。
- `http://127.0.0.1:8080/health` 返回成功响应，Web 页面可访问。
- 重启 Compose 后数据目录仍存在，固定 JWT/TOTP 密钥未改变。
- Git 暂存区和提交历史不包含 `.env`、运行数据目录或生成的管理员密码。
- `main` 分支成功推送到 `0zero-start/own-sub2api`，并保留 `upstream` 官方远程。
