# Hajimi 应用 Docker 部署指南 🚀

## 欢迎使用 Hajimi 应用！本指南将引导您使用 Docker 在不同环境（本地电脑、远程服务器、NAS）中快速部署 Hajimi 应用。

## 1. 准备工作 🛠️

### 1.1 安装 Docker
请确保您的系统（Windows, macOS, Linux）已安装 Docker。
- **Windows/Linux 用户**：推荐安装 Docker Desktop。
- **macOS 用户**：除了 Docker Desktop 外，也可以考虑使用 [OrbStack](https://orbstack.dev/download) (一款高性能的 Docker Desktop 替代品)。

- **下载地址**：[Docker 官方网站](https://www.docker.com/get-started)
- **安装步骤**：按照官方指引完成安装。

### 1.2 获取配置文件
下载部署所需的两个核心配置文件：

**[.env文件下载](https://github.com/beijixingxing/hajimi/blob/main/wiki/docker/.env)**

**[docker-compose.yaml文件下载](https://github.com/beijixingxing/hajimi/blob/main/wiki/docker/docker-compose.yaml)**

### 1.3 创建项目目录并放置配置文件 📂
在您选择的位置创建一个专门用于部署 Hajimi 的文件夹，并将下载的 `.env` 和 `docker-compose.yaml` 文件放入该文件夹中。
- **Mac/Linux**：在终端执行 `mkdir ~/hajimi-app && cd ~/hajimi-app`
- **Windows**：
    - 在命令提示符执行 `mkdir C:\Users\<用户名>\hajimi-app && cd C:\Users\<用户名>\hajimi-app` （请将 `<用户名>` 替换为您的 Windows 用户名）
    - 或在 PowerShell 执行 `mkdir $HOME\hajimi-app; cd $HOME\hajimi-app`
或者，您也可以直接在桌面或其他位置创建 `hajimi-app` 文件夹，然后将下载的两个文件移动进去。

**重要提示**：后续的所有命令行操作，除非特别说明，都应在此 `hajimi-app` 文件夹内执行。

### 1.4 通用配置步骤 ⚙️
在启动服务之前，您需要根据自己的需求修改配置文件。请使用文本编辑器打开 `hajimi-app` 文件夹中的 `.env` 和 `docker-compose.yaml` 文件。

#### 1.4.1 修改 .env 文件
这是存放敏感信息和应用基本配置的地方。
```dotenv
# --- 必填项 ---
# 设置你的 Gemini API 密钥，可以设置多个，用英文逗号隔开
GEMINI_API_KEYS=key1,key2,key3  # <= 替换为你的真实密钥

# 设置应用的登录密码
PASSWORD=123   # <= 设置一个安全的密码，默认密码123，请务必修改！

# --- 可选项 (Vertex AI) ---
# 是否启用 Google Vertex AI (默认关闭)
ENABLE_VERTEX=false           # <= 如需开启，改为 true

# 如果开启 Vertex AI，请填入完整的 Google Cloud 服务账号 JSON 密钥内容
# 注意：整个 JSON 内容需要包含在英文单引号 ' ' 内部
GOOGLE_CREDENTIALS_JSON='JSON 密钥内容' # <= 粘贴你的 JSON 密钥内容
```
**提示**：`.env` 文件中还包含更多高级配置项，您可以根据末尾的 “附录：参考环境变量详情” 按需调整。

#### 1.4.2 按需修改 docker-compose.yaml 文件
这个文件定义了 Docker 如何运行 Hajimi 服务。主要关注以下几点：

*   **端口映射**：如果 `7860` 端口已被其他应用占用，您需要修改端口映射。
*   **网络代理**：如果您的部署环境无法直接访问 Google API (或其他所需外部服务)，需要配置代理。
*   **数据持久化 (重要！)**：默认配置已启用数据持久化。
*   **时区**：默认配置为中国时区。

```yaml
services:
  hajimi-app: # 服务名，可自定义
    image: beijixingxing/hajimi:latest
    container_name: hajimi-app  # 固定容器名方便监控
    labels:
      - "com.centurylinklabs.watchtower.enable=true"  # 启用自动更新
    ports:
      - "7860:7860" # 端口映射：冒号左边是宿主机端口（可改），右边是容器内端口（固定7860）。
                    # 如果 7860 端口冲突，修改冒号左边的数字，例如 "17860:7860"
    env_file:
      - .env  # 加载 .env 文件中的环境变量
    environment:
      - TZ=Asia/Shanghai # 时区配置，默认中国时区。如需修改，例如：America/New_York
      - ENABLE_STORAGE=true # 应用层面的数据持久化开关，配合 volumes 使用

      # --- 网络代理 (按需取消注释并修改) ---
      # 如果需要 HTTP 代理:
      # HTTP_PROXY: "http://your_proxy_address:port"  # <= 例如 "http://127.0.0.1:7890"
      # 如果需要 HTTPS 代理 (通常与 HTTP 代理相同):
      # HTTPS_PROXY: "http://your_proxy_address:port" # <= 例如 "http://127.0.0.1:7890"
      # 注意：在 Windows/Mac 的 Docker Desktop 环境下，若代理在宿主机上，可尝试使用 "host.docker.internal"作为地址，
      # 例如: HTTP_PROXY: "http://host.docker.internal:7890"
      # 在 Linux 服务器上，请直接填写代理服务器的 IP 地址和端口。

    volumes: # 数据持久化配置
      - ./settings:/hajimi/settings # 将容器内的 /hajimi/settings 目录映射到当前项目文件夹下的 settings 子目录。
                                   # 这会保存应用的所有重要配置和数据，即使容器更新或重启也不会丢失。
                                   # 首次启动后，您会在 hajimi-app 文件夹下看到一个新创建的 settings 文件夹。
    restart: unless-stopped # 容器退出时自动重启，除非是手动停止的。

  # 🆙 自动更新监控服务（默认每小时检查一次）
  watchtower:
    image: containrrr/watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock  # 必须的Docker连接配置
    environment:
      - TZ=Asia/Shanghai # 时区配置
      - WATCHTOWER_LABEL_ENABLE=true  # 只监控带有 "com.centurylinklabs.watchtower.enable=true" 标签的容器
      - WATCHTOWER_POLL_INTERVAL=3600  # 检查间隔秒数（3600秒 = 1小时）
      - WATCHTOWER_CLEANUP=true  # 更新后自动删除旧镜像
    restart: unless-stopped
```
配置完成后，请保存文件。

## 2. 选择部署方式并启动服务 ▶️
根据您的环境选择相应的部署方式：

### 2.1 本地电脑部署 (Docker Desktop / OrbStack)
1.  打开终端 (Mac/Linux) 或命令提示符/PowerShell (Windows)。
2.  进入项目目录：使用 `cd` 命令切换到您之前创建的 `hajimi-app` 文件夹。
    *   **示例 (Mac/Linux)**：`cd ~/hajimi-app`
    *   **示例 (Windows CMD)**：`cd C:\Users\<用户名>\hajimi-app`
    *   **示例 (Windows PowerShell)**：`cd $HOME\hajimi-app`
3.  启动服务：执行以下命令以后台模式启动服务。
    ```bash
    docker compose up -d
    ```
    Docker 会自动拉取镜像并根据 `docker-compose.yaml` 和 `.env` 文件启动容器。

### 2.2 远程服务器部署 (SSH)
1.  使用 SSH 工具（如 `ssh` 命令、PuTTY、Termius 等）连接到您的远程服务器。
2.  创建并进入项目目录：
    ```bash
    # 创建目录 (路径可自定义) 并进入该目录
    mkdir -p /path/to/your/hajimi-app && cd /path/to/your/hajimi-app
    # 例如: mkdir -p /opt/hajimi-app && cd /opt/hajimi-app
    ```
3.  直接下载配置文件：
    ```bash
    # 下载 .env 文件模板
    wget https://raw.githubusercontent.com/beijixingxing/hajimi/main/wiki/docker/.env

    # 下载 docker-compose.yaml 文件
    wget -O docker-compose.yaml https://raw.githubusercontent.com/beijixingxing/hajimi/main/wiki/docker/docker-compose.yaml
    ```
    或使用 `curl`：
    ```bash
    curl -o .env https://raw.githubusercontent.com/beijixingxing/hajimi/main/wiki/docker/.env
    curl -o docker-compose.yaml https://raw.githubusercontent.com/beijixingxing/hajimi/main/wiki/docker/docker-compose.yaml
    ```
4.  **重要：编辑配置文件**
    使用文本编辑器（如 `nano` 或 `vim`）修改下载的 `.env` 文件和 `docker-compose.yaml` 文件。
    *   **必须修改 `.env` 文件**：
        ```bash
        nano .env
        ```
        至少需要修改 `GEMINI_API_KEYS` 和 `PASSWORD`。根据需要调整其他配置。
    *   **按需修改 `docker-compose.yaml` 文件**（例如端口、代理）：
        ```bash
        nano docker-compose.yaml
        ```
        参考 “1.4.2 按需修改 docker-compose.yaml 文件” 部分的说明。
5.  验证文件：
    ```bash
    ls -l  # 应看到 .env 和 docker-compose.yaml 两个文件
    cat .env # 检查 .env 配置文件内容是否已按需修改
    ```
6.  启动服务：在 SSH 终端中，确保您位于项目目录下，执行启动命令。
    ```bash
    docker compose up -d
    ```

### 2.3 NAS 部署 (通过 Docker Compose UI)
注意：不同 NAS 品牌的 Docker UI（如群晖的 Container Manager, QNAP 的 Container Station）操作略有不同，以下为通用步骤。
1.  通过 NAS 的文件管理工具（如 File Station）在 Docker 应用的数据存储区（通常是 `/volume1/docker/` 或类似路径）创建一个 `hajimi` 或 `hajimi-app` 文件夹。
2.  将您在本地电脑上按照 “1.4 通用配置步骤” **修改好**的 `.env` 和 `docker-compose.yaml` 文件上传到 NAS 上刚创建的文件夹中。
3.  打开 NAS 的 Docker 管理应用（如 Container Manager）。
4.  寻找 “项目” (Project)、“应用” (Application) 或 “Compose” 相关的选项。
5.  选择 “创建” (Create) 或 “导入” (Import)。
6.  设置项目名称（如 `hajimi`），并选择已上传 `docker-compose.yaml` 文件所在的文件夹路径。
7.  系统通常会自动识别 `docker-compose.yaml` 文件。确认配置无误后，点击 “创建”、“部署” 或 “运行”。
8.  NAS 的 Docker UI 会根据配置文件拉取镜像并启动容器。

## 3. 访问与验证 ✅
服务启动后，稍等片刻让应用完全启动。
1.  **本地访问**：打开浏览器，访问 `http://localhost:7860`（如果您修改了端口，请使用修改后的端口）。
2.  **服务器/NAS 访问**：打开浏览器，访问 `http://<服务器或NAS的IP地址>:7860`（请将 `<服务器或NAS的IP地址>` 替换为实际 IP，端口同样根据配置修改）。
3.  **API 端点**：应用的 API (兼容 OpenAI 格式) 可以在以下地址访问：`http://<访问地址>:7860/v1`

看到登录界面并能使用您在 `.env` 文件中设置的 `PASSWORD` 成功登录，即表示部署成功！

## 4. 常见问题与解决 (FAQ) ❓

### 4.1 Q1: 启动时提示端口已被占用 (Port is already allocated)
-   **原因**：`7860` 端口（或其他您配置的端口）已被系统上另一个程序使用。
-   **排查**：
    -   **Mac/Linux**：在终端运行 `sudo lsof -i :7860`
    -   **Windows**：在命令提示符或 PowerShell 运行 `netstat -ano | findstr "7860"`
-   **解决方案**：
    -   停止占用该端口的程序。
    -   或者，修改 `hajimi-app` 文件夹中 `docker-compose.yaml` 文件里的 `ports` 部分，将冒号左侧的 `7860` 改为其他未被占用的端口（如 `17860`），例如：`ports: - "17860:7860"`。保存后需要重新启动服务 (`docker compose down` 然后 `docker compose up -d`)。

### 4.2 Q2: 应用无法连接外部服务 (如 Google API)
-   **原因**：部署环境的网络无法直接访问所需服务，通常需要设置网络代理。
-   **解决方案**：
    -   **确认代理**：确保您有可用的 HTTP/HTTPS 代理服务器地址和端口。
    -   **配置代理**：编辑 `hajimi-app` 文件夹中的 `docker-compose.yaml` 文件，在 `hajimi-app` 服务的 `environment` 部分取消 `HTTP_PROXY` 和 `HTTPS_PROXY` 的注释 (`#`)，并填入您的代理地址。例如:
        ```yaml
        environment:
          # ... 其他环境变量 ...
          HTTP_PROXY: "http://192.168.1.100:7890"
          HTTPS_PROXY: "http://192.168.1.100:7890"
          # 如果代理在宿主机上 (Docker Desktop for Win/Mac), 可尝试:
          # HTTP_PROXY: "http://host.docker.internal:7890"
          # HTTPS_PROXY: "http://host.docker.internal:7890"
        ```
    -   **重启服务**：保存文件后，在项目目录下执行 `docker compose down` 然后 `docker compose up -d`。
    -   **无需代理**：如果您的网络环境不需要代理，请确保 `HTTP_PROXY` 和 `HTTPS_PROXY` 配置被注释掉（前面有 `#`）或直接删除。

## 5. 更新指南 🔄

### 5.1 自动更新 (内置)
`docker-compose.yaml` 文件中已包含 Watchtower 服务，用于自动检测并更新 Hajimi 应用的 Docker 镜像。默认设置是每小时检查一次。如果检测到新版本，它会自动拉取并重启容器。

### 5.2 手动更新
如果您想立即更新到最新版本或禁用了自动更新，可以按以下步骤手动更新：
1.  进入项目目录：使用 `cd` 命令切换到包含 `docker-compose.yaml` 文件的 `hajimi-app` 目录。
    ```bash
    # 示例 (路径需替换为您的实际路径)
    cd /path/to/your/hajimi-app
    ```
2.  拉取最新镜像：
    ```bash
    docker compose pull hajimi-app # 或者直接 docker compose pull，会拉取所有服务的最新镜像
    ```
    这将只拉取 `hajimi-app` 服务的新镜像版本（如果存在）。
3.  停止并重新创建容器：
    ```bash
    # 停止当前运行的容器
    docker compose down
    # 使用新镜像重新创建并启动容器
    docker compose up -d
    ```
    或者，更简洁的方式是直接执行 `docker compose up -d --pull`，Compose 会先尝试拉取新镜像，然后如果镜像有更新，会自动重新创建容器。

4.  **可选的强制清理命令（仅在遇到问题时使用）**：
    ```bash
    # 警告：以下命令会停止并删除容器、网络。 --rmi all 会删除服务相关的镜像。
    # 请谨慎操作，并确保您了解其影响。
    # docker compose down --rmi all
    
    # 然后重新拉取并启动
    # docker compose pull
    # docker compose up -d
    ```

**建议**：首次部署时，尽量使用默认配置（除了必要的 API 密钥和密码），确保服务能正常运行。稳定运行后，再根据需要调整端口、代理或其他高级配置。

---

## 附录：参考配置文件详情

以下是 `.env` 和 `docker-compose.yaml` 文件的参考内容，其中包含更多可配置项的说明。在实际部署中，您主要通过修改 `hajimi-app` 目录下的这两个文件来进行配置。

### A.1 参考环境变量详情 (.env 文件)
```dotenv
# --- 🌟 基础安全配置 ---
# 访问密码，用于访问服务的身份验证 (必填，请修改为强密码)
PASSWORD=123
# Web UI修改设置密码，如果留空，则默认值为 PASSWORD 的值
WEB_PASSWORD=

# --- ⏰ 时区配置 ---
# (此项通常在 docker-compose.yaml 中配置，此处列出供参考，但 .env 文件中的 TZ 不会被 Hajimi 应用直接使用)
# TZ=Asia/Shanghai

# --- 🤖 AI Studio核心配置 ---
# 用英文逗号分隔多个API KEY，可使用多个Gemini API密钥 (必填)
GEMINI_API_KEYS=key1,key2,key3
# 每分钟最多请求次数，限制系统每分钟接收的请求数量
MAX_REQUESTS_PER_MINUTE=30
# 每天每个IP的请求上限，防止单个 IP 过度请求
MAX_REQUESTS_PER_DAY_PER_IP=600
# 假装实时传输，用于解决某些情况下客户端断开连接的问题
FAKE_STREAMING=true
# 每个KEY每天最多用100次，限制单个 API 密钥在一天内的使用次数
API_KEY_DAILY_LIMIT=100
# 生成迷惑字符串，可在转发消息中添加随机字符串伪装消息，防止被检测
RANDOM_STRING=true
# 迷惑字符串长度，随机字符串的长度
RANDOM_STRING_LENGTH=5
# 默认并发请求数，系统默认同时处理的请求数量
CONCURRENT_REQUESTS=1
# 请求失败时增加的并发请求数，请求失败后增加的并发处理数量
INCREASE_CONCURRENT_ON_FAILURE=0
# 允许的最大并发请求数，系统允许同时处理的最大请求数量
MAX_CONCURRENT_REQUESTS=3
# 是否启用联网模式，决定是否使用联网搜索功能
SEARCH_MODE=false
# 联网模式提示词，在联网搜索时的提示信息
SEARCH_PROMPT='（使用搜索工具联网搜索，需要在content中结合搜索内容）'
# 需要屏蔽的模型名称，多个模型用英文逗号分隔
BLOCKED_MODELS=
# 空响应重试次数，当请求返回空响应时的重试次数
MAX_EMPTY_RESPONSES=5

# --- 📋 白名单配置 ---
# 模型白名单，仅允许列表中的模型通过，多个用英文逗号分隔 (留空则不限制)
WHITELIST_MODELS=
# 请求头User-Agent白名单模式，仅允许特定的User-Agent访问 (留空则不限制)
WHITELIST_USER_AGENT=

# --- 📝 缓存配置 ---
# 切换缓存计算方法，默认为 false (使用快速但不精确的缓存键)，true 表示使用精确但稍慢的缓存键
PRECISE_CACHE=false

# 📦 持久化配置（已集成在compose文件里）
# 持久化存储目录，默认为 /hajimi/settings/
STORAGE_DIR=/hajimi/settings/
# 是否启用持久化，默认为 false
ENABLE_STORAGE=false

# --- 🔑 Vertex高级配置 ---
# 是否启用vertex，决定是否使用Vertex AI服务，默认关闭
ENABLE_VERTEX=false
# 填入完整的Google凭证JSON (base64编码或直接JSON字符串)，用于访问Vertex AI服务的凭证
# 如果直接使用JSON字符串，请确保其被英文单引号包裹且内部引号已转义，或考虑将其作为 Docker Secret 管理
GOOGLE_CREDENTIALS_JSON=''
```

### A.2 参考 docker-compose.yaml 文件内容
(此内容已在教程主体部分的 "1.4.2 按需修改 docker-compose.yaml 文件" 中详细展示和解释，此处不再重复，请参考上文。)
