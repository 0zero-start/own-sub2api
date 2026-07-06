# Docker Compose 项目改名设计

## 目标

将 Docker Desktop 中当前显示为 `deploy` 的本地 Compose 项目改名为
`own-sub2api`，不改变容器名称、端口、管理员账号或持久化数据。

## 实现

在已被 Git 忽略的 `deploy/.env` 中设置：

```dotenv
COMPOSE_PROJECT_NAME=own-sub2api
```

先显式以旧项目名 `deploy` 执行 `docker compose down`，只删除旧项目的容器和网络；
不使用 `--volumes`，也不删除 `deploy/data`、`deploy/postgres_data` 或
`deploy/redis_data`。随后使用同一 Compose 文件和 `.env` 重新启动。绑定目录保持不变，
因此 PostgreSQL、Redis 和应用数据继续使用原位置。

## 验证

- Docker Compose 项目列表显示 `own-sub2api`，不再显示 `deploy`。
- `sub2api`、`sub2api-postgres`、`sub2api-redis` 均为健康状态。
- `http://127.0.0.1:8080/health` 和首页均返回 HTTP 200。
- `deploy/.env` 与三个运行数据目录仍未被 Git 跟踪。
