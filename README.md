# ACR Mirror

使用 GitHub Actions + skopeo 将 Docker 镜像全架构同步到阿里云容器镜像服务（ACR），供国内服务器使用。

## 特性

- ✅ 支持 DockerHub、gcr.io、k8s.io、ghcr.io 等任意镜像源
- ✅ 自动同步所有架构（amd64/arm64/...），`docker pull` 自动匹配
- ✅ 不落盘、不装 Docker，skopeo 直接流式传输
- ✅ 支持大型镜像（无磁盘空间限制）
- ✅ 自动处理镜像重名（不同命名空间的同名镜像）
- ✅ 支持定时同步

## 工作原理

```
DockerHub/GCR ──skopeo copy --all──▶ 阿里云 ACR
```

利用 GitHub Actions 的海外 runner 作为中转，skopeo 直接将镜像从源 registry 流式传输到阿里云 ACR，保留完整的 manifest list。国内服务器拉取时自动匹配架构。

## 使用方式

### 1. 配置阿里云 ACR

登录 [阿里云容器镜像服务](https://cr.console.aliyun.com/)

- 启用个人实例（或企业实例）
- 创建一个命名空间 → 记为 `ALIYUN_NAME_SPACE`
- 访问凭证 → 获取用户名 `ALIYUN_REGISTRY_USER` 和密码 `ALIYUN_REGISTRY_PASSWORD`
- 仓库地址 → 记为 `ALIYUN_REGISTRY`（如 `registry.cn-hangzhou.aliyuncs.com`）

### 2. Fork 本项目

Fork 后进入 **Settings → Secrets and variables → Actions → New Repository secret**

添加以下 4 个 Secret：

| Secret 名称 | 值 | 示例 |
|-------------|---|------|
| `ALIYUN_REGISTRY` | 阿里云仓库地址 | `registry.cn-hangzhou.aliyuncs.com` |
| `ALIYUN_NAME_SPACE` | 命名空间 | `my-images` |
| `ALIYUN_REGISTRY_USER` | 用户名 | `user@example.com` |
| `ALIYUN_REGISTRY_PASSWORD` | 密码 | `********` |

### 3. 添加镜像

编辑 `images.txt`，每行一个镜像地址：

```
# 简单镜像
nginx:latest
alpine:3.19

# 带命名空间
xhofe/alist:latest
fatedier/frps:v0.61.2

# 私有仓库
k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.0.0
registry.k8s.io/ingress-nginx/controller:v1.12.1

# AI/ML 镜像
vllm/vllm-openai:v0.8.5
```

提交后自动触发 GitHub Actions 同步。

### 4. 使用镜像

在国内服务器拉取：

```bash
docker pull registry.cn-hangzhou.aliyuncs.com/<命名空间>/nginx:latest
```

Docker 会根据机器架构自动选择对应的镜像（amd64/arm64/...）。

## 重名处理

如果 `images.txt` 中有同名镜像（不同命名空间），会自动加命名空间前缀：

```
xhofe/alist      → registry.xxx/my-space/xhofe_alist:latest
xiaoyaliu/alist  → registry.xxx/my-space/xiaoyaliu_alist:latest
```

## 定时执行

修改 `.github/workflows/docker.yaml`，添加 schedule：

```yaml
on:
  workflow_dispatch:
  push:
    branches: [main]
  schedule:
    - cron: '0 0 * * 1'  # 每周一 UTC 00:00
```

## 技术栈

- **skopeo**：容器镜像操作工具，支持跨 registry 复制、不落盘
- **GitHub Actions**：免费 CI/CD，海外 runner 天然可访问 DockerHub/GCR

## License

MIT
