# lorca-relay-image

为 [egoist/lorca](https://github.com/egoist/lorca) 的 relay（`lorca-relay`）自动构建的多架构 Docker 镜像：**linux/amd64 + linux/arm64**（树莓派 5 可直接拉取 arm64）。

上游仓库只提供 relay 的 Dockerfile、不发布预编译镜像。本仓库在检测到上游发布新的 CLI 版本（`cli-v*` tag；relay 跟随同一 workspace 版本号）时自动构建，并推送到 GHCR。

## 镜像

```
ghcr.io/lee88688/lorca-relay:<版本>   # 例如 0.1.11
ghcr.io/lee88688/lorca-relay:latest   # 最新版本
```

## 运行

```bash
docker run -d --name lorca-relay \
  -p 8787:8787 \
  -v lorca-relay:/data \
  -e LORCA_RELAY_SECRET=$(openssl rand -hex 32) \
  ghcr.io/lee88688/lorca-relay:latest
```

检查状态：

```bash
curl http://localhost:8787/v1/health
# {"ok":true,"service":"lorca-relay"}
```

然后在 Lorca 客户端里把 Relay URL 指向你的服务地址（公网部署需要反代 HTTPS 并支持 WebSocket 升级，同时设置 `LORCA_RELAY_TRUST_PROXY=true`，让限流拿到真实客户端 IP）。完整的环境变量说明见上游 [crates/relay/README.md](https://github.com/egoist/lorca/blob/main/crates/relay/README.md)。

## docker compose

```yaml
services:
  relay:
    image: ghcr.io/lee88688/lorca-relay:latest
    restart: unless-stopped
    ports:
      - "8787:8787"
    volumes:
      - relay-data:/data
    environment:
      # 换成 openssl rand -hex 32 的输出；不设的话每次重启都会使所有 Device 的 token 失效
      LORCA_RELAY_SECRET: "请替换"

volumes:
  relay-data:
```

## 构建机制

- **定时检查**：每 6 小时通过 `git ls-remote` 查询上游最新的 `cli-v*` tag（不消耗 API 配额），发现新版本即构建
- **原生双架构构建**：amd64 跑在 `ubuntu-latest`，arm64 跑在原生 `ubuntu-24.04-arm` runner 上，不用 QEMU 模拟，Rust 编译速度快得多
- **manifest 合并**：两个单架构镜像（`:<版本>-amd64`、`:<版本>-arm64`）构建完成后合并为一个 multi-arch manifest，并更新 `latest` 标签
- **幂等**：已包含双架构的镜像自动跳过；可在 Actions 页面手动触发（workflow_dispatch），支持指定上游 tag 和强制重建

## 注意

- **版本号跟随上游 workspace 版本**（即 CLI 版本），relay 本身没有独立的版本号或 tag
- 仓库 **60 天没有任何 commit** 时，GitHub 会暂停定时任务；手动运行一次 workflow 或推一个 commit 即可恢复
- 树莓派 5 需要运行 **64 位系统**（arm64）；32 位系统无法运行该镜像
- relay 与客户端之间的兼容性由 relay 协议版本决定（`/v1/health` 可查），选 tag 时以客户端年代为准
