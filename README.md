# Sync binaries of kubectl

从 Kubernetes 官网 <https://dl.k8s.io/VERSION/TGZ_FILENAME> 同步 kubectl(及可选 server/node)二进制程序,并通过 GitHub Actions 自动发布到本仓库的 Releases.

## 工作原理

同步流程由本仓库 `.github/workflows/dosync.yaml` 触发,实际同步逻辑实现于共享组合 Action `abldgsync/actions/kubectl`(其 `dosync.sh` 按 `CS=1~4` 阶段执行);本仓库工作流仅负责调用与传参。核心步骤如下:

1. **解析版本并生成下载清单**
   - 未指定版本时使用 GitHub API 的 `latest` 接口获取最新稳定版;指定版本时通过 `tags/<version>` 接口精确命中.
   - 若解析到预发布版本(`rc`/`beta`/`alpha`)则直接报错退出,避免误发非稳定版.
   - **下载清单取自官方 SHA512SUMS**:直接拉取 `dl.k8s.io/<ver>/SHA512SUMS`,它末尾列出了各 `kubernetes-*.tar.gz` 包名及其 sha512 校验值.从中筛选 tar.gz 条目(默认仅 `kubernetes.tar.gz` + `kubernetes-client-*`,开启 `withsvr` 时含 server/node),下载 URL 由 `https://dl.k8s.io/<ver>/<文件名>` 直接组装,无需写死平台列表,且天然适配各版本架构.
2. **并行下载全部二进制包**:基于 `xargs -P` 并发下载(默认 8 路并发,失败自动重试).
3. **校验完整性**:使用 SHA512SUMS 中解析出的 sha512 校验值,通过 `sha512sum -c` 逐文件校验.
4. **发布到 Releases**:以版本号为 tag(如 `v1.33.6`),上传所有下载文件.

> 调用 GitHub API 时已携带 `GITHUB_TOKEN` 鉴权,将限流从 60 次/小时提升到 5000 次/小时.

## 触发方式

- **定时触发**:`schedule` 已预留(每月 10 号凌晨 2 点 `0 2 10 * *`,默认注释),取消注释即可启用,使用最新稳定版.
- **手动触发**(`workflow_dispatch`):可在 Actions 页面手动运行,支持以下输入参数:

| 参数 | 说明 | 必填 | 示例 |
| --- | --- | --- | --- |
| `binvern` | 指定版本号(如 `v1.33.6` 或 `1.33.6`),不填则使用最新稳定版 | 否 | `v1.33.6` |
| `withsvr` | 是否包含 SERVER 及 NODE 二进制包,取值 `0`/`1`,默认 `0` | 否 | `1` |

## 产物

每次同步会在 Releases 中生成一个以版本号命名的发行(如 `Kubectl v1.33.6 Binaries`),包含对应平台的二进制包(`kubernetes-client-*.tar.gz` 等),可直接下载使用.

## 目录结构

```
kubectl/
├── .github/workflows/
│   └── dosync.yaml   # 同步工作流定义
├── LICENSE
└── README.md
```
