# nginx-uos-multiarch

UOS Server 基础镜像 + nginx 官方 stable 版本的多架构（`linux/amd64` + `linux/arm64`）镜像构建仓库，托管在 GitHub，用 Actions 在 nginx 官方发布 stable 新版本时自动构建。

## 仓库结构

```
.
├── .github/workflows/docker_build_and_publish_uos.yml  # 版本检测 + 分平台构建 + manifest 合并 + 打 tag
├── nginx-stable/                        # 变体 1：nginx-stable（x86 + arm64 合并为一个 Dockerfile）
│   ├── Dockerfile
│   └── default.conf
└── nginx-stable-otel/                   # 变体 2：nginx-stable-otel（含 otel/njs/image-filter/xslt 模块）
    ├── Dockerfile
    ├── default.conf
    └── nginx.conf
```

## 产物（共 2 套镜像 × 2 架构）

镜像仓库：`ghcr.io/blueapple168/nginx`

| 变体 | 最终 tag | 浮动 tag | 平台 |
|---|---|---|---|
| stable | `<版本>-uos`（如 `1.30.6-uos`） | `latest` | linux/amd64, linux/arm64 |
| otel | `<版本>-otel-uos`（如 `1.30.6-otel-uos`） | `otel-latest` | linux/amd64, linux/arm64 |

构建过程会先推送单架构临时 tag（`<最终tag>-amd64` / `<最终tag>-arm64`），再由 merge job 用 `docker buildx imagetools create` 合成 multi-arch manifest；`docker pull` / `docker run` 按宿主机架构自动选择。

基础镜像：`ghcr.io/blueapple168/uos-server-20-1070a:latest`（UOS Server 20，RHEL 8 兼容，glibc 2.28，需同时含 amd64/arm64 架构）。

## 版本触发机制

1. Workflow 每天 `02:00 UTC` 定时运行，抓取 `https://nginx.org/en/download.html` 解析 **Stable version** 区块的最新版本号；
2. 与仓库中的 git tag `nginx-v<版本>` 比对——不存在则认为有新版本，触发构建；
3. 构建成功后自动打 tag `nginx-v<版本>`，下次不再重复构建。

手动触发：Actions → *Docker Build and Publish (UOS nginx)* → *Run workflow*
- `nginx_version`：指定版本（如 `1.30.6`），留空自动检测；
- `force_build`：勾选后跳过 tag 检查，同版本强制重建并覆盖镜像（改了 Dockerfile/配置后用这个）。

## Workflow 流程（4 个 job）

| Job | 作用 |
|---|---|
| `detect` | 解析 nginx 最新 stable 版本，比对 git tag 决定是否构建 |
| `build-and-publish` | matrix：`variant(stable/otel) × platform(amd64/arm64)` = 4 个独立 job；amd64→`ubuntu-latest`，arm64→`ubuntu-24.04-arm` 原生 runner（public 仓库免费，不用 QEMU）；推单架构 tag 后跑 Trivy 扫描（出报告不阻断） |
| `merge-manifest` | 校验两个 arch tag 都存在后，`imagetools create` 合成 multi-arch manifest 到版本 tag + 浮动 tag；`always()` 保证单变体失败不影响另一变体 |
| `tag-release` | 构建成功后打 `nginx-v<版本>`，完成版本闭环 |

## 可配置项

| 位置 | 名称 | 默认 | 说明 |
|---|---|---|---|
| workflow env | `IMAGE_REPO` | `ghcr.io/blueapple168/nginx` | 镜像推送仓库 |
| workflow env / Dockerfile ARG | `OS_VERSION` | `20-1070a` | UOS 基础镜像版本（仅用于 label/构建参数） |
| job if | `github.repository_owner` | `blueapple168` | 防 fork 触发；可换成数字 ID `github.repository_id == '<ID>'`（改名免疫） |

无额外 Secrets 需求：基础镜像与目标镜像同在 ghcr.io，`GITHUB_TOKEN` 登录即可拉取/推送（基础镜像 package 需对本仓库可见）。

## 前置要求 / 注意事项

- **基础镜像架构**：`ghcr.io/blueapple168/uos-server-20-1070a:latest` 必须同时包含 `amd64` 与 `arm64` 两种架构，否则 arm64 job 会因 manifest 缺失失败。
- **arm64 原生 runner**：`ubuntu-24.04-arm` 对 public 仓库免费且不限量；private 仓库需付费或改回 QEMU 模拟（把 runner 映射改掉并补 QEMU 步骤）。
- **仓库可见性**：若仓库为 private，`ubuntu-24.04-arm` 不可免费用，需评估费用或退回 QEMU 方案。
- **GHCR 首次推送**后，包默认为 private，需到 GitHub Packages 设置里按需调整可见性（内网拉取需能访问 ghcr.io）。
- otel 镜像的 `nginx.conf` 默认上报端点 `${OTEL_ENDPOINT:-otel-collector:4317}`，可用环境变量 `OTEL_ENDPOINT` / `OTEL_TOKEN` / `OTEL_SERVICE_NAME` 覆盖。
- Actions 引用中 `docker/setup-buildx-action`、`docker/login-action`、`lhotari/sandboxed-trivy-action` 已按参考 workflow 用 commit SHA 锁定；`actions/checkout@v5`、`actions/upload-artifact@v4` 建议后续也换成 SHA 锁定。

## 快速开始

```bash
git init && git add . && git commit -m "init: uos nginx multiarch"
git remote add origin git@github.com:blueapple168/<repo>.git
git push -u origin main
# 到 Actions 页手动 Run workflow 一次验证（可勾选 force_build），之后即全自动
```
