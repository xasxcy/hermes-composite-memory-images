# hermes-composite-memory-images

GitHub Actions 构建 Hermes 复合记忆后端的自定义镜像，推送到 GHCR。NAS 只 `docker pull @digest`，不再本地构建。

起因：NAS 是孱弱且网络受限的构建环境（代理限速），Mem0 镜像连续暴露 grpcio 断流、`[graph]` no-op、`psycopg` 缺 libpq 三类构建缺陷。GH runner 外网干净、约 3 分钟完成、免代理。方案详见 vault `dispatch/2026-07-13_dual-memory-hermes-plan/PLAN.md` 附录 G。

## 镜像

| 目录 | 镜像 | tag | 说明 |
|---|---|---|---|
| `mem0/` | `ghcr.io/xasxcy/hermes-mem0` | `2.2.1-94c3fe9f` | Mem0 self-hosted REST server，base digest 钉住 + 源码 commit 锁定 |
| `honcho/` | `ghcr.io/xasxcy/hermes-honcho` | `3.5.0-0cd37ddb-uv0.13.8-tiktoken1` | Honcho API + Deriver 共用镜像，官方 commit 锁定并在构建期验证上游 JWT NumericDate 行为 |

Graphiti（Gate 3）将来再加一个目录 + 一个 workflow job。

## 构建来源真源

`mem0/Dockerfile` 是本仓库 CI 的构建源，必须与 vault 部署仓库 `hermes-composite-memory/deploy/mem0/Dockerfile` **逐字一致**（该仓库有 `tests/test_mem0_deploy_contract.py` 校验构建契约）。改任一处后两边同步并复核 SHA-256。当前 SHA（2026-09-27）：`e46099dac756ecc60316b1992e432b975b6c2c390a64844d351632561aa3aa1a`。

镜像不 bake 任何秘密：runtime 的 `.env`（DB 密码、API key、JWT）全部在 NAS 侧注入，不进镜像也不进本仓库。

## Honcho JWT 验证

Honcho `v3.5.0`（`0cd37ddbd88f51cdac7c660d7372f4b2fd40a4b3`）沿用上游原生 JWT NumericDate，因此旧维护 patch 已退休。Dockerfile 在构建期用上游真实代码签发两枚 token：未来 `exp` 必须验证通过，过去 `exp` 必须抛 `AuthenticationException("JWT expired")`；上游的 NumericDate 往返或过期语义回归会使构建失败（2026-10-18 已用 v3.3.0 源码实测并做过反例对照；2026-11-01 用 uv 0.9.24 在 v3.5.0 上 frozen 安装并复跑同一探针，PROBE OK）。回退使用 NAS 上同一时间点创建的 `hermes-honcho:rollback-<timestamp>` 本地标签，或上一个已验证的固定 digest。

## NAS 拉取

当前镜像 digest（Honcho 为 v3.5.0；Mem0 为 2026-09-27 的构建）：
- Honcho（当前，v3.5.0 + uv 0.13.8 + 构建期预置 tiktoken 词表）: `sha256:cbae7f0a99540091a56fd4bb36870360cd0b37f9857f3dce7bfb89f4fa93b3de`（`0cd37ddb`，tag `3.5.0-0cd37ddb-uv0.13.8-tiktoken1`；2026-11-15 GH Actions run 43672343368 构建，Dockerfile 带 `RUN --network=none` 断言，NAS 同日上线，Image ID `85eb9ecb2544`，容器断网可 import）。
- Honcho（uv 0.13.8、未预置词表的上一版，可直接回退）: `sha256:b5fa2270277b335737f0ed5fc851ab6afd08ee6f34cce72d15b951296052546e`（tag `3.5.0-0cd37ddb-uv0.13.8`，本地锚点 `rollback-20261115T113310`，Image ID `d3375f418949`；2026-11-14 至 11-15 生产在跑；2026-11-14 GH Actions run 43548067436 构建，基础镜像 `ghcr.io/astral-sh/uv:0.13.8@sha256:5cc1805e…bac85`；NAS 同日上线，api 与 deriver 同一镜像，Image ID `d3375f418949`）。
- Honcho（v3.5.0 + uv 0.9.24，上一版，可直接回退：应用 commit 与 schema 均未变）: `sha256:ca541957ac6e7ba08c6d5f1588d33263054f6d3dc00a30a917808d8d322571b1`（tag `3.5.0-0cd37ddb`，NAS 本地锚点 `hermes-honcho:rollback-20261114T105354`，Image ID `cd53a6741094`；2026-11-01 至 11-14 生产在跑）。注意：迁移执行后只改 tag 回退 3.3.0 会因 alembic 不认新 revision 而启动失败。
- Honcho（v3.3.0 上一版，仅作参考，迁移后不能直接回退）: `sha256:e7e8ace457ec64c38fcfb3ea2b63dd64bb28ec778490da3d05442ba5a1bd300d`（v3.3.0，`608aec8a`；2026-10-18 GH Actions run 39130365424 构建，NAS `docker pull` 后 RepoDigest 一致，OCI `image.revision` 标签 = 冻结 SHA；当前生产在跑）
- Mem0: `sha256:4c06fbd2328ec831a776775a58bcaa4f3d95fb4df7d1fd3799e6c347f138617e`

本仓库私有 → GHCR 包默认私有，NAS 拉取需先登录：

```sh
# 1) 登录 GHCR（在 NAS root 环境；PAT 需 read:packages 权限，用 stdin 传入不落历史）
echo "<GHCR_PAT>" | sudo -i docker login ghcr.io -u xasxcy --password-stdin

# 2) 按 digest 拉取（比 tag 更可复现）
sudo -i docker pull ghcr.io/xasxcy/hermes-mem0@sha256:4c06fbd2328ec831a776775a58bcaa4f3d95fb4df7d1fd3799e6c347f138617e

# 3) 打回本地 tag 供 compose 使用（compose 里镜像名保持 hermes-mem0:2.2.1-94c3fe9f）
sudo -i docker tag ghcr.io/xasxcy/hermes-mem0@sha256:4c06fbd2328ec831a776775a58bcaa4f3d95fb4df7d1fd3799e6c347f138617e hermes-mem0:2.2.1-94c3fe9f
```

若把 GHCR **包**（非仓库）设为 public，则跳过第 1 步免登录直接拉。之后 `cd /volume2/docker/hermes-composite-memory/mem0 && sudo -i docker compose up -d` 会直接用这个本地 tag，不再构建。
