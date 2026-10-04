# docker-venv

调用 GitHub Actions Runner 的 **Docker Image Builder**：在仓库 Variables 中传入完整 Dockerfile（直接贴全文，或给一个下载 URL），外部文件由 `pre_build` 命令自备，手动一键触发，多镜像并发构建并推送到阿里云 ACR。**全程免 commit、免开 PR。**

---

## 📁 目录结构

```text
.
├── .github/workflows/
│   └── docker-image-builder.yml   # 唯一入口：校验矩阵 → 解析 Dockerfile → 构建推送
├── .dockerignore                  # 仅排除 .git（构建上下文为仓库根目录时）
├── .gitignore
└── README.md
```

---

## 🚀 使用方式

配置好 Variables 与 Secrets（见下文）后，三种触发方式任选：

| 方式 | 操作 |
| :--- | :--- |
| **网页** | 仓库 **Actions** 标签 → 选中 `docker-image-builder` → 右侧 **Run workflow** → 点绿色按钮，立即执行，无表单输入 |
| **GitHub CLI** | `gh workflow run docker-image-builder.yml` |
| **REST API** | `POST /repos/{owner}/{repo}/actions/workflows/docker-image-builder.yml/dispatches` |

配置全部来自仓库 Variables——想换构建内容，去 Settings 改 JSON，再点一次 Run workflow 即可，不需要改任何仓库文件。

### 构建流程

```
Run workflow
   │
   ▼
prepare  读取 vars.BUILD_MATRIX → 逐项硬校验 → 生成动态矩阵
   │
   ▼
build    对每个矩阵项并发执行：
   1. checkout 触发仓库（提供默认工作区文件）
   2. 展开 secrets.SECRET_ENV（可选：逐条注册掩码 → 注入后续步骤环境 + 落盘为 buildx secret）
   3. 执行 pre_build 自定义命令（可选：clone 仓库、下载解压归档、备齐 COPY 所需文件）
   4. 按 dockerfile.mode 解析出完整 Dockerfile，写入 context 目录（inline 解码 / url 下载）
   5. 登录 ACR（仅 push=true）
   6. docker buildx 在 context 目录中构建 → 推送 registry/namespace/name:tag
```

---

## 🔐 GitHub 云端配置

`Settings` → `Secrets and variables` → `Actions`

### 1. 密钥 (Secrets)

| 名称 | 说明 |
| :--- | :--- |
| `ACR_USERNAME` | 阿里云容器镜像服务登录账号 |
| `ACR_PASSWORD` | 阿里云容器镜像服务固定访问凭证 |
| `SECRET_ENV` | ⬜ **聚合保密环境变量**，多行 `KEY=value`（见「🔑 敏感参数」）。增删条目只改这里，无需 commit |

### 2. 变量 (Variables)

| 名称 | 必填 | 说明 |
| :--- | :--- | :--- |
| `ACR_REGISTRY` | ✅ | ACR 实例地址（如 `crpi-xxxx.cn-guangzhou.personal.cr.aliyuncs.com`） |
| `ACR_NAMESPACE` | ✅ | ACR 命名空间（如 `jiahui-sync`） |
| `BUILD_MATRIX` | ✅ | 镜像构建矩阵，JSON 数组，字段规范见下文 |
| `BUILD_TIMEOUT_MINUTES` | ⬜ | 单镜像构建超时，默认 `60` |

> **已废弃**：旧版的 `BASE_CONFIG`、`OVERRIDE_CONFIG`、`BUILD_SCHEMA` 不再被读取，可以到 Variables 列表中删除。
>
> ⚠️ `vars.*` 的内容会**原样出现在构建日志**中，任何凭证（密码、token、密钥）**一律放 Secrets**，绝不能写进 Variables。
> 想要"可任意增删、且不需要 commit 的保密环境变量"，全部写进聚合 Secret `SECRET_ENV`（见「🔑 敏感参数」）。

---

## 🧩 BUILD_MATRIX 字段参考

`BUILD_MATRIX` 是一个 **JSON 数组**，每个元素是一个独立的镜像构建任务：

```json
[
  {
    "name": "my-app",
    "tag": "v1",
    "dockerfile": { "mode": "inline", "content": ["FROM ubuntu:24.04", "RUN ..."] },
    "context": "可选，构建工作目录（默认仓库根 .）",
    "pre_build": "可选，构建前执行的命令数组（每元素一行）",
    "build_args": { "可选": "构建参数键值对" },
    "push": true,
    "platforms": "linux/amd64"
  }
]
```

### 顶层字段

| 字段 | 类型 | 必填 | 默认值 | 说明 |
| :--- | :--- | :--- | --- | :--- |
| `name` | string | ✅ | — | 镜像名，匹配 `^(?!-)[a-z0-9._-]+$` |
| `tag` | string | ✅ | — | 镜像标签，匹配 `^[a-zA-Z0-9._-]+$` |
| `dockerfile` | object | ✅ | — | Dockerfile 来源，见下表 |
| `context` | string | ⬜ | `.` | **构建工作目录**（相对仓库根的子目录）：解析出的 Dockerfile 写入此目录，`docker buildx` 也在此目录中执行（`COPY` 引用的文件相对它）。目录须已存在——不存在时应由 `pre_build` 创建 |
| `pre_build` | array | ⬜ | 无 | 构建前执行的命令，**JSON 字符串数组：每个元素一行、末尾带逗号**（在 runner 上按顺序拼成 bash 脚本执行）：clone 仓库、下载/解压文件、安装工具等，一切备料都靠它。与 `content` 同样遵守单行铁律 |
| `build_args` | object | ⬜ | `{}` | 任意 `ARG` 键值对，全量透传给 `docker build --build-arg` |
| `push` | bool | ⬜ | `true` | `false` 时仅构建验证，不登录也不推送（镜像不会导出到本地） |
| `platforms` | string | ⬜ | `linux/amd64` | 构建平台，如 `linux/amd64,linux/arm64` |

### `dockerfile` 字段（两模式互斥，都提供**完整 Dockerfile**）

| `mode` | 专属字段 | 说明 |
| :--- | :--- | :--- |
| `inline` | `content` | 完整 Dockerfile，**JSON 字符串数组：每个元素一行、末尾带逗号**；空行写空字符串 `""` |
| `url` | `url` | 完整 Dockerfile 的 http(s) 下载地址（如 raw.githubusercontent.com） |

> **单行铁律**：`content` 数组的每个元素必须是**一整行**，元素内不得含换行符。跨多行的 `RUN ... && \` 长指令请按 Dockerfile 续行符拆成多个元素（`\` 结尾一行，续行再一个元素）。构建时按顺序以换行拼回完整 Dockerfile。

解析出的 Dockerfile 统一写入 `context` 目录下的 `Dockerfile.generated` 并以它构建。若想使用某个 Git 仓库自带的 Dockerfile，用 `pre_build` clone 该仓库，再以 `mode=url` 指向其 raw 地址（或直接 inline 内容），并把 `context` 指向 clone 出的目录。

### 示例

**① inline——直接从 JSON 传入完整 Dockerfile（每元素一行）**

```json
[
  {
    "name": "hello",
    "tag": "v1",
    "dockerfile": {
      "mode": "inline",
      "content": [
        "FROM ubuntu:24.04",
        "RUN apt-get update && \\",
        "    apt-get install -y curl",
        "",
        "CMD [\"echo\", \"hello\"]"
      ]
    }
  }
]
```

**② url——从 URL 拉取 Dockerfile**

```json
[
  {
    "name": "hello",
    "tag": "v2",
    "dockerfile": {
      "mode": "url",
      "url": "https://raw.githubusercontent.com/owner/repo/main/Dockerfile"
    }
  }
]
```

**③ pre_build clone 仓库 + context——用外部 Git 仓库作为构建上下文**

```json
[
  {
    "name": "caddy",
    "tag": "v1",
    "dockerfile": {
      "mode": "url",
      "url": "https://raw.githubusercontent.com/caddyserver/caddy/master/Dockerfile"
    },
    "context": "caddy-src",
    "pre_build": ["git clone --depth 1 https://github.com/caddyserver/caddy.git caddy-src"]
  }
]
```

**④ pre_build + build_args——构建前备料 + 传构建参数**

```json
[
  {
    "name": "my-app",
    "tag": "v1",
    "dockerfile": {
      "mode": "inline",
      "content": [
        "FROM ubuntu:24.04",
        "COPY files/config.toml /etc/app/config.toml",
        "ARG APP_VERSION",
        "RUN echo ${APP_VERSION}"
      ]
    },
    "pre_build": [
      "mkdir -p files",
      "curl -fsSL https://example.com/config.toml -o files/config.toml"
    ],
    "build_args": { "APP_VERSION": "1.2.3" },
    "push": true,
    "platforms": "linux/amd64"
  }
]
```

**⑤ 多镜像矩阵 + 仅构建不推送**

```json
[
  { "name": "app", "tag": "stable", "dockerfile": { "mode": "url", "url": "https://example.com/Dockerfile.stable" } },
  { "name": "app", "tag": "edge",   "dockerfile": { "mode": "url", "url": "https://example.com/Dockerfile.edge" }, "push": false }
]
```

---

## 📦 构建上下文与外部文件

`COPY` / `ADD` 的源**必须是构建上下文内的本地文件**。构建上下文由 `context` 字段决定——默认 `.`（触发 workflow 时 checkout 的本仓库根目录），解析出的 Dockerfile 也会写进该目录，`docker buildx` 在其中执行。

需要往上下文里加外部文件（克隆其他仓库、下载解压归档、生成配置……）时：

1. **`pre_build` 一把梭**（主力手段，构建前执行任意命令，做完备料再解析 Dockerfile、开始构建）：
   ```json
   "pre_build": [
     "git clone --depth 1 https://github.com/owner/repo.git src",
     "curl -fsSL https://example.com/docker-venv-files.zip -o /tmp/f.zip",
     "python3 -m zipfile -e /tmp/f.zip ."
   ]
   ```
   - clone 其他仓库当上下文 → 再配 `"context": "src"` 指向 clone 目录
   - 下载 zip 解压到仓库根 → 沿用默认 context，Dockerfile 里 `COPY docker-venv-files/...`
2. **Dockerfile 内自取**：`ADD https://example.com/x /opt/x` 或 `RUN curl -fsSL https://example.com/x -o /opt/x`（文件在镜像层内生成，不经过上下文）

> 执行顺序：`checkout` → 展开 `SECRET_ENV`（如已配置）→ `pre_build` → 解析 Dockerfile 写入 `context` → 构建。`context` 目录必须在 `pre_build` 结束时**已存在**，否则构建前报错中止（默认 `.` 永远存在）。

---

## 🔑 敏感参数：聚合 Secret `SECRET_ENV`

需要为构建引入**保密**的环境变量（私有下载地址、token、第三方仓库凭证……）时，
**不要**新增独立 Secret 再去改 workflow（那要 commit）。统一写进一个多行 Secret：

`Settings` → `Secrets and variables` → `Actions` → 新建 Secret，名为 `SECRET_ENV`，内容每行一个 `KEY=value`：

```ini
# 私有归档的完整下载地址（可含签名参数，或 user:token@ 凭证——值本身会被掩码）
PRIVATE_URL=https://private.example.com/files/docker-venv-files.zip
GIT_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx
NPM_TOKEN=npm_xxxxxxxxxxxxxxxxxxxx
```

**增删改条目只编辑这个 Secret，永远不需要 commit。** build job 的 `Load secret env block` 步骤会在 `pre_build` 之前展开它，同时打通两条引用通道。

### 通道 1：`pre_build` 中引用（runner 上的命令）

展开后每个键都进入后续步骤的环境变量，直接用 shell 变量（注意是 `$NAME`，**不是** `${{ }}`）：

```json
"pre_build": [
  "curl -fsSL --retry 3 \"$PRIVATE_URL\" -o /tmp/docker-venv-files.zip",
  "python3 -m zipfile -e /tmp/docker-venv-files.zip ."
]
```

需要 Bearer 头的写法：

```json
"pre_build": [
  "curl -fsSL --retry 3 -H 'Authorization: Bearer '$GIT_TOKEN \"$PRIVATE_URL\" -o /tmp/docker-venv-files.zip",
  "python3 -m zipfile -e /tmp/docker-venv-files.zip ."
]
```

> ⚠️ `pre_build` 以 `bash -e -u -o pipefail` 执行：引用了 `SECRET_ENV` 里**不存在**的键会因 `set -u` 直接失败中止。要么保证配置齐全，要么写成 `${GIT_TOKEN:-}` 给默认值。

私有 Git 仓库当上下文也一样：

```json
"pre_build": [
  "git clone --depth 1 https://x-access-token:${GIT_TOKEN}@github.com/owner/private-repo.git src"
]
```
（再配 `"context": "src"`）

### 通道 2：Dockerfile 中引用（构建期）

整份键值对会作为 BuildKit secret 挂载（**id 固定为 `SECRET_ENV`**），不进镜像层、历史和缓存：

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=SECRET_ENV sh -c '\
      set -a; . /run/secrets/SECRET_ENV; set +a; \
      curl -fsSL -H "Authorization: Bearer $GIT_TOKEN" "$PRIVATE_URL" -o /tmp/payload.bin'
```

> - 挂载文件里每行是 `KEY='value'`（值自动加单引号、内部单引号已转义），所以 `source` 时空格、`$(...)`、反引号等都不会被 shell 解析执行；`pre_build` 通过环境变量拿到的是**原值**（无引号）。
> - `--mount` 依赖 BuildKit（本项目已用 buildx），建议 Dockerfile 首行加 `# syntax=docker/dockerfile:1`。
> - 未配置 `SECRET_ENV`（或无有效条目）时不会挂载该 secret，引用它的 `RUN` 会失败——**Dockerfile 与 Secret 要配套**。
> - 首选仍是「`pre_build` 备料 + `COPY`」：能让 Dockerfile 不碰敏感值，就不必开这条通道。

### 格式规则与限制

| 项 | 规定 |
| :--- | :--- |
| 行格式 | `KEY=value`；`#` 开头视为注释；空行/纯空白行忽略；行尾 `CR` 自动剥离 |
| 键名 | 必须匹配 `^[A-Za-z_][A-Za-z0-9_]*$`；禁用会破坏 runner 的键：`PATH`、`HOME`、`IFS`、`PWD`、`OLDPWD`、`SHELL`、`ENV`、`BASH_ENV`、`PROMPT_COMMAND`、`LD_PRELOAD`、`LD_LIBRARY_PATH`、`TMPDIR`、`GITHUB_*`、`RUNNER_*`、`INPUT_*` |
| 值 | 首尾空白自动去除；引号**不做** shell 解析（原样保留）；**不支持多行值**（一行一个条目）；落盘为 buildx secret 文件时逐行自动加单引号 |
| 日志掩码 | 每个值逐条注册 `::add-mask::`，此后在日志中出现即变 `***`；值为空或长度 <4 时**不注册**（仅 warning，避免短字符串把整份日志打成 `***`） |
| 失败行为 | 某行缺 `=`、键名不合法或命中禁用键 → 立即 `::error::` 中止构建（报错只给行号和键名，不回显值） |

> ⚠️ **为什么要自己注册掩码**：GitHub 对多行 Secret 是按**整行**（`KEY=value`）打码的，单独 `echo $KEY` 只输出值时不会命中掩码。这一步会逐值补 `::add-mask::`，所以值本身安全；但把值**拼进更长的字符串**（如带 token 的 URL 被 `curl`/`git` 失败时回显）在注册前仍可能明文出现，别在 `pre_build` 里开 `set -x`。

---

## 🧠 变量机制要点（必读）

1. **Variables 的值不会被表达式求值**：在 Variables 页面填 `${{ secrets.XXX }}` 只会得到字面量字符串，**不会展开**。vars 之间也不能互相引用。
2. **需要凭证时一律用 Secrets**：workflow 已把 `ACR_USERNAME` / `ACR_PASSWORD` 注入 `pre_build` 步骤的环境变量，脚本里直接用 `$ACR_USERNAME`、`$ACR_PASSWORD` 即可（GitHub 会对日志中的 secret 值自动打码为 `***`）。**其他保密环境变量写进聚合 Secret `SECRET_ENV`（见上一节），无需改 workflow、无需 commit。**
3. **动态引用 secret 名**（进阶）：vars 里存 secret 的**名字**，在 workflow 表达式中用索引语法取值：
   ```yaml
   env:
     TOKEN: ${{ secrets[vars.MY_TOKEN_NAME] }}
   ```
4. **为什么需要聚合 Secret**：Actions 表达式**没有循环**，`env:` 的**键名必须静态写在 YAML 里**——所以"仓库里有几个 Secret 就自动生成几个同名环境变量"在表达式层做不到。`Load secret env block` 步骤把这一步下移到 bash：workflow 只留一行静态注入，条目数量由 Secret 内容决定，增删条目因此不需要 commit。

---

## ⚠️ 约束与安全

- **体量上限**：单个 GitHub Variable 限 **48 KB**——超长 Dockerfile 请用 `mode=url`；整个矩阵输出受 job output **1 MB** 限制（多镜像 + 大量 inline 时留意）。
- **pre_build 是在 runner 上执行任意命令**：能修改仓库 Variables 的人（需 write 权限）等同于能修改 workflow 本身；`workflow_dispatch` 仅仓库成员可触发，fork PR 无法触发。**绝不要把凭证写进 `pre_build` 或任何 Variable**。
- **pre_build 克隆私有仓库/访问私有地址需要凭证**：一律写进聚合 Secret `SECRET_ENV`（见「🔑 敏感参数」），脚本里用 `$NAME` 引用；**不要**把 token 直接写进 `pre_build` 或任何 Variable（那就是明文进日志）。
- **敏感值绝不进 `build_args`**：`build_args` 来自明文 Variables，且 `--build-arg` 的值会**留在镜像历史**（`docker history` 可见）。构建期确实要用敏感值，走 `SECRET_ENV` 的 secret mount。
- **`SECRET_ENV` 的可见范围是整个 build job**（展开后进 `GITHUB_ENV`，并落盘成 secret 文件供挂载）：只放构建真正需要的条目，别当密码保险库用。GitHub 托管 runner 每次全新、任务后销毁。
- **push=false 只做构建验证**：镜像留在 BuildKit 缓存中，**不会**导出到 runner 本地 docker，也不会推送。
- url 模式的 Dockerfile 下载失败、产物缺少 `FROM` 指令、`context` 目录不存在、`pre_build` 命令失败，均在构建前报错中止。

---

## 🔧 校验规则

prepare 阶段对 `BUILD_MATRIX` 逐项硬校验，任何一项不合法即中止并精确报告位置（`vars.BUILD_MATRIX[i]`）：

- JSON 必须是非空数组，每项是对象
- `name` / `tag` 正则匹配；未知键打印 ⚠️ 警告并忽略
- `dockerfile.mode ∈ {inline, url}`；`inline` 的 `content` 必须是非空字符串数组（元素为单行、拼接后含 `FROM`）；`url` 必须 `http(s)://` 开头
- `context` 必须是单行工作区内相对路径（不得以 `/` 开头或含 `..`）
- `pre_build` 必须是非空字符串数组（元素单行、拼接后至少一条有效命令）；`build_args` 键匹配 `^[A-Za-z_][A-Za-z0-9_]*$`、值为标量且不含换行
- `push` 必须是 JSON 布尔值；`platforms` 单行非空

`SECRET_ENV`（聚合保密变量）在 build 阶段的 `Load secret env block` 步骤逐行校验：某行缺 `=`、键名不合法或命中禁用键名 → 报错中止；值为空或长度 <4 → 仅警告不注册掩码。未设置该 Secret 时整步跳过，不影响构建。
