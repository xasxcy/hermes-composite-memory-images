# hermes-composite-memory-images

GitHub Actions 构建 Hermes 复合记忆后端的自定义镜像，推送到 GHCR。NAS 只 `docker pull @digest`，不再本地构建。

起因：NAS 是孱弱且网络受限的构建环境（代理限速），Mem0 镜像连续暴露 grpcio 断流、`[graph]` no-op、`psycopg` 缺 libpq 三类构建缺陷。GH runner 外网干净、约 3 分钟完成、免代理。方案详见 vault `dispatch/2026-07-13_dual-memory-hermes-plan/PLAN.md` 附录 G。

## 镜像

| 目录 | 镜像 | tag | 说明 |
|---|---|---|---|
| `mem0/` | `ghcr.io/xasxcy/hermes-mem0` | `2.0.12-42cf18c4` | Mem0 self-hosted REST server，base digest 钉住 + 源码 commit 锁定 |

Honcho（Gate 2）、Graphiti（Gate 3）将来各加一个目录 + 一个 workflow job。

## 构建来源真源

`mem0/Dockerfile` 是本仓库 CI 的构建源，必须与 vault 部署仓库 `hermes-composite-memory/deploy/mem0/Dockerfile` **逐字一致**（该仓库有 `tests/test_mem0_deploy_contract.py` 校验构建契约）。改任一处后两边同步并复核 SHA-256。当前 SHA：`9d3d76a4a22f69bf9a9f08e32fcabb0ecc3460b2afc72ef0cee2d16da038f28d`。

镜像不 bake 任何秘密：runtime 的 `.env`（DB 密码、API key、JWT）全部在 NAS 侧注入，不进镜像也不进本仓库。

## NAS 拉取

当前 Mem0 镜像 digest（首建 2026-07-17）：
`sha256:eb1bc4099a9c396a94762e6d71ec5c5cb725347210748897696b60d25467bac8`

本仓库私有 → GHCR 包默认私有，NAS 拉取需先登录：

```sh
# 1) 登录 GHCR（在 NAS root 环境；PAT 需 read:packages 权限，用 stdin 传入不落历史）
echo "<GHCR_PAT>" | sudo -i docker login ghcr.io -u xasxcy --password-stdin

# 2) 按 digest 拉取（比 tag 更可复现）
sudo -i docker pull ghcr.io/xasxcy/hermes-mem0@sha256:eb1bc4099a9c396a94762e6d71ec5c5cb725347210748897696b60d25467bac8

# 3) 打回本地 tag 供 compose 使用（compose 里镜像名保持 hermes-mem0:2.0.12-42cf18c4）
sudo -i docker tag ghcr.io/xasxcy/hermes-mem0@sha256:eb1bc4099a9c396a94762e6d71ec5c5cb725347210748897696b60d25467bac8 hermes-mem0:2.0.12-42cf18c4
```

若把 GHCR **包**（非仓库）设为 public，则跳过第 1 步免登录直接拉。之后 `cd /volume2/docker/hermes-composite-memory/mem0 && sudo -i docker compose up -d` 会直接用这个本地 tag，不再构建。
