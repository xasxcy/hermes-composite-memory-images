# hermes-composite-memory-images

GitHub Actions 构建 Hermes 复合记忆后端的自定义镜像，推送到 GHCR。NAS 只 `docker pull @digest`，不再本地构建。

起因：NAS 是孱弱且网络受限的构建环境（代理限速），Mem0 镜像连续暴露 grpcio 断流、`[graph]` no-op、`psycopg` 缺 libpq 三类构建缺陷。GH runner 外网干净、约 3 分钟完成、免代理。方案详见 vault `dispatch/2026-07-13_dual-memory-hermes-plan/PLAN.md` 附录 G。

## 镜像

| 目录 | 镜像 | tag | 说明 |
|---|---|---|---|
| `mem0/` | `ghcr.io/xasxcy/hermes-mem0` | `2.0.19-19cb89af` | Mem0 self-hosted REST server，base digest 钉住 + 源码 commit 锁定 |
| `honcho/` | `ghcr.io/xasxcy/hermes-honcho` | `3.1.0-82a92429-jwtfix1` | Honcho API + Deriver 共用镜像，官方 commit 锁定并应用单一 JWT NumericDate 维护 patch |

Graphiti（Gate 3）将来再加一个目录 + 一个 workflow job。

## 构建来源真源

`mem0/Dockerfile` 是本仓库 CI 的构建源，必须与 vault 部署仓库 `hermes-composite-memory/deploy/mem0/Dockerfile` **逐字一致**（该仓库有 `tests/test_mem0_deploy_contract.py` 校验构建契约）。改任一处后两边同步并复核 SHA-256。当前 SHA（2026-08-31）：`89e296765dc2e01b84896374fb82923329a68e61d9de55f348d6c7fc6dd02e53`。

镜像不 bake 任何秘密：runtime 的 `.env`（DB 密码、API key、JWT）全部在 NAS 侧注入，不进镜像也不进本仓库。

## Honcho 维护 patch

`honcho/patches/0001-jwt-expiry-numericdate.patch` 目前钉在官方 commit `82a92429b888727b2236820b863256067c7edc80`（2026-08-31 升级 3.0.12→3.1.0，`git apply --check` 实测通过，三块以行偏移 +15/+15/-4 干净套用，`src/security.py` 关键符号 create_jwt/verify_jwt/JWTParams 均在）：将 JWT `exp` 正规化为 RFC 7519 NumericDate，修复 ISO string 被 PyJWT 拒绝、numeric value 又被服务端二次 ISO 解析的缺陷。Dockerfile 在构建期先 `git apply --check`，再签发/验证一个短期 numeric-exp token；任一上游上下文漂移或语义回归都会 fail build。回退只需把 NAS Compose 的 `HONCHO_IMAGE_TAG` 改回上一个已验证 tag（本轮之前是 `3.0.12-44489797-jwtfix1`，回滚锚点见 vault `UPGRADE-MANIFEST.md`）并使用对应固定 digest 的本地 tag。

## NAS 拉取

当前镜像 digest（2026-08-31 升级，待 GH Actions 构建后回填）：
- Honcho: `sha256:258b06eed5e3b6086e3d53dc00e203f386eaa8c50960e187f6e15c5825bcc2c3`
- Mem0: `sha256:396cbd7a157c4c2f671784aaaa06d675eba788cfcba68dab9818880988e233ab`

本仓库私有 → GHCR 包默认私有，NAS 拉取需先登录：

```sh
# 1) 登录 GHCR（在 NAS root 环境；PAT 需 read:packages 权限，用 stdin 传入不落历史）
echo "<GHCR_PAT>" | sudo -i docker login ghcr.io -u xasxcy --password-stdin

# 2) 按 digest 拉取（比 tag 更可复现）
sudo -i docker pull ghcr.io/xasxcy/hermes-mem0@sha256:396cbd7a157c4c2f671784aaaa06d675eba788cfcba68dab9818880988e233ab

# 3) 打回本地 tag 供 compose 使用（compose 里镜像名保持 hermes-mem0:2.0.19-19cb89af）
sudo -i docker tag ghcr.io/xasxcy/hermes-mem0@sha256:396cbd7a157c4c2f671784aaaa06d675eba788cfcba68dab9818880988e233ab hermes-mem0:2.0.19-19cb89af
```

若把 GHCR **包**（非仓库）设为 public，则跳过第 1 步免登录直接拉。之后 `cd /volume2/docker/hermes-composite-memory/mem0 && sudo -i docker compose up -d` 会直接用这个本地 tag，不再构建。
