# GitHub 整目录上传（github_upload）

Operit 沙盒插件。把本地目录**一次性提交**到 GitHub 仓库：走 Git Data API（blobs → tree → commit → 更新引用），整个目录只产生 **one commit**，而不是每个文件一次 PUT。

适合上传整包源码、素材目录、构建产物；不适合用它管理 Issue / PR / 分支评审 —— 那些用内置的 `github` 包。

## 安装

两种方式任选：

1. **插件市场**：在 Operit 里搜索「GitHub 整目录上传」直接安装。
2. **手动导入**：从 [Releases](https://github.com/huoxingrNaNa/OperitForge/releases) 下载最新的 `github-upload-v*.js`，放进 `Android/data/com.ai.assistance.operit/files/packages/`，重启软件或在包管理里刷新。

| 项 | 值 |
| --- | --- |
| 包名 | `github_upload` |
| 当前版本 | 0.1.2 |
| 最低软件版本 | 1.12.2（实测 `versionName 1.12.2` / `versionCode 51`） |
| 最高软件版本 | 1.99.99 |
| 工具数 | 6 |

## 工具

| 工具 | 作用 |
| --- | --- |
| `ghu_check` | 检查令牌：登录名、scopes、剩余配额。**上传前先跑这个** |
| `ghu_repo_info` | 仓库信息：默认分支、私有与否、体积、权限、网页地址 |
| `ghu_create_repo` | 创建仓库（默认私有，可选 `org`、`auto_init`） |
| `ghu_push_file` | 上传/更新**单个**文件：`local_path` 走本地读（二进制安全），或 `content` 直接写文本（UTF-8 自动 base64）；已存在自动取 sha 覆盖；分支不存在自动基于默认分支创建 |
| `ghu_push_dir` | 上传**整个目录**，合并成一次提交；默认增量合并，`mirror=true` 才镜像（先删远端多余文件）；`include` / `exclude` 支持通配且区分大小写 |
| `ghu_netcheck` | 连通性自检：`api.github.com` / `raw.githubusercontent.com` / `uploads.github.com` 的耗时与状态，并给出是否需要代理、超时怎么调的结论 |

典型流程：

```text
ghu_check                                    # 令牌与配额
→ ghu_netcheck                               # 网络差时先跑
→ ghu_create_repo                            # 需要新仓库时
→ ghu_push_dir(local_dir="/sdcard/Download/proj", remote_dir="src")
```

## 令牌配置

插件按顺序找令牌，**只认显式配置，不猜任何固定路径**（0.1.2 起移除了旧版对 `/root/.gh_token` 的隐式读取）：

1. 环境变量 `GITHUB_TOKEN`
2. 环境变量 `GITHUB_TOKEN_FILE` 指向的文件（内容可以是纯令牌，也可以是 `GITHUB_TOKEN=xxx` 整行）

两者都不设置时**不会读取任何文件**，并直接报错提示怎么配。候选令牌会逐个探活，第一个可用的才缓存，所以环境变量里残留了失效令牌也不会卡住整包。

> 不要把令牌写进插件源码，也不要把令牌提交到本仓库。

令牌权限：私有仓库读写需要 `repo`；要建/删仓库需要 `delete_repo`；要改 `.github/workflows` 需要 `workflow`。

### 本机设置示例（Linux / proot）

```bash
# 方式一：直接给环境变量
export GITHUB_TOKEN=ghp_xxx

# 方式二：从文件读
echo 'ghp_xxx' > ~/.gh_token
export GITHUB_TOKEN_FILE=~/.gh_token
```

在 Operit 里也可以用环境变量写入工具设置 `GITHUB_TOKEN_FILE=/root/.gh_token`。

## 与内置 `github` 包的区别

内置 `github` 包（20 个工具）覆盖仓库、Issue、PR、文件提交、分支、差异提交。本插件不重复这些，独有的是：

| 能力 | 说明 |
| --- | --- |
| **整目录一次 commit** | 内置包逐文件 `PUT /contents/{path}`，N 个文件 N 次提交；本插件用 Git Data API 合并成一次 |
| **镜像模式** | `mirror=true` + `remote_dir`，先清远端多余文件再写，适合“本地为准”的同步；禁止镜像仓库根目录 |
| **令牌探活** | 多来源候选逐个验证，失效会说明“已试过哪些来源” |
| **网络诊断** | `ghu_netcheck` 分别测三个域名的连通与耗时，给出超时/代理建议 |
| **二进制直传** | 手写 base64（沙盒里没有 `btoa`），无需先把文件转成文本 |

## 已知边界

- 目录上传默认最多 200 个文件（`max_files` 可调）。GitHub 单次 tree 提交的文件数有服务端上限。
- 默认排除 `.git`、`node_modules`、`build`、`.gradle`。
- `include` / `exclude` 的通配匹配区分大小写。
- 校验用的是实际请求返回码：只有 404 才视为“文件不存在”，403 / 429 会如实报错（不会被误判成可新建）。

## 许可

MIT，见 [LICENSE](LICENSE)。

作者：huoxingrNaNa
