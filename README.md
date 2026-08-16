# Docker Images Pusher

使用 GitHub Actions + skopeo 将 Docker 镜像全架构同步到阿里云私有仓库，供国内服务器使用。

- ✅ 支持 DockerHub、gcr.io、k8s.io、ghcr.io 等任意仓库
- ✅ 自动同步所有架构（amd64/arm64/...），`docker pull` 自动匹配
- ✅ 不落盘、不装 Docker，skopeo 直接流式传输
- ✅ 支持最大 40GB+ 的大型镜像
- ✅ 自动处理镜像重名

视频教程：https://www.bilibili.com/video/BV1Zn4y19743/

作者：**[技术爬爬虾](https://github.com/tech-shrimp/me)**
B站，抖音，Youtube 全网同名，转载请注明作者

---

## 改进说明（skopeo 版本）

本项目基于 [tech-shrimp/docker_image_pusher](https://github.com/tech-shrimp/docker_image_pusher) 改进，主要变化：

| 原版 | 改进版 |
|------|--------|
| docker pull → tag → push | skopeo copy 一步到位 |
| 需要 Docker daemon | 不需要，skopeo 独立运行 |
| 镜像落盘，需清理磁盘 | 不落盘，流式传输 |
| 需手动指定 `--platform`，前缀区分 | `--all` 全架构，manifest list 自动匹配 |
| 磁盘空间折腾（删 .NET/Haskell） | 不需要，无磁盘压力 |

---

## 使用方式

### 1. 配置阿里云

登录 [阿里云容器镜像服务](https://cr.console.aliyun.com/)

- 启用个人实例
- 创建一个命名空间 → 记为 **ALIYUN_NAME_SPACE**
- 访问凭证 → 获取用户名 **ALIYUN_REGISTRY_USER** 和密码 **ALIYUN_REGISTRY_PASSWORD**
- 仓库地址 → 记为 **ALIYUN_REGISTRY**（如 `registry.cn-hangzhou.aliyuncs.com`）

### 2. Fork 本项目

Fork 后进入 Settings → Secrets and variables → Actions → New Repository secret

添加以下 4 个 Secret：

| Secret 名称 | 值 |
|-------------|---|
| `ALIYUN_REGISTRY` | 阿里云仓库地址 |
| `ALIYUN_NAME_SPACE` | 命名空间 |
| `ALIYUN_REGISTRY_USER` | 用户名 |
| `ALIYUN_REGISTRY_PASSWORD` | 密码 |

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
```

提交后自动触发 GitHub Actions 同步。

### 4. 使用镜像

在国内服务器拉取：

```bash
docker pull registry.cn-hangzhou.aliyuncs.com/<命名空间>/nginx:latest
```

Docker 会根据机器架构自动选择对应的镜像（amd64/arm64/...）。

---

## 重名处理

如果 `images.txt` 中有同名镜像（不同命名空间），会自动加命名空间前缀：

```
xhofe/alist      → registry.xxx/my-space/xhofe_alist:latest
xiaoyaliu/alist  → registry.xxx/my-space/xiaoyaliu_alist:latest
```

---

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
