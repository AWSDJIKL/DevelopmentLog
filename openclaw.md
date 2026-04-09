# OpenClaw部署相关记录

## 前言

部署需求总结：
- 多名用户同时使用，因此应建立大量对应agent作为区分。
- 鉴于token太贵，需使用本地部署大模型，宿主机的PRO 6000拥有96GB显存，利用ollama框架可以勉强部署qwen3.5-122B模型（约占91GB显存）。
- 尽量agent避免对各类环境文件的改写权限，包括且不限于系统级文件、conda虚拟环境等。在skill的选择和编写上应注意。


考虑生产环境安全性需求，经简单调研后，选择使用docke部署。对比直接在物理机上部署，有以下优点：

- **虚拟网关方便管理**
- **有官方教程较为便捷**
- **一旦出事方便清理**：容器挂载结构清晰，一旦发生系统崩溃或被攻击，可随时删档重启。

但同样有以下难点：
- **越权风险**：docker后台服务是sudo权限，因此尽管可以限制容器没有sudo权限，但依然可以通过`docker run -v /home/other_user:/mnt -it ubuntu bash`或`docker run -v /:/host -it ubuntu bash`等命令直接挂在其他用户甚至系统文件进行读写，有越权风险。（考虑使用Rootless Docker规避风险）

20260402：先以官方教程跑通一个实例，后续考虑使用Rootless Docker限制权限。

## 部署过程中遇到的问题和解决

总体部署流程参照[官方教程](https://docs.openclaw.ai/install/docker)，下文仅记录遇到的问题和解决思路

宿主机中仅部署一个openclaw，其中创建多个agent按用户或功能区分，避免上下文混乱。

1. 部署openclaw的用户应分配到docker组，否则无法执行docker命令，创建openclaw用户专门部署。
    ```
    # 在具有sudo权限的用户终端中执行，用于部署的用户以openclaw为例
    # 创建用户
    sudo adduser openclaw
    # 添加到docker组
    sudo usermod -aG docker openclaw
    ```

2. openclaw容器访问宿主机ollama服务

    首先需要让宿主机的ollama服务绑定到0.0.0.0:11434，即监听本机IP的11434端口的所有IP的访问。原本默认是仅监听来自127.0.0.1的访问，而docker容器与宿主机来自不同网关，无法直接连接。
    ```
    # 创建配置文件夹
    sudo mkdir -p /etc/systemd/system/ollama.service.d
    # 创建配置文件并写入
    echo -e "[Service]\nEnvironment="OLLAMA_HOST=0.0.0.0"" | sudo tee /etc/systemd/system/ollama.service.d/override.conf
    # 重新加载服务配置
    sudo systemctl daemon-reload
    # 重新启动ollama
    sudo systemctl restart ollama
    ```
    最后在配置ollama base URL是填入`http://{宿主机的局域网IP}:11434`即可。

3. 局域网内其他的物理机无法访问openclaw的网关仪表盘，显示`origin not allowed (open the Control UI from the gateway host or allow it in gateway.controlUi.allowedOrigins)`
    
    **方法一：** 修改配置，开放访问权限，使得局域网内可以以非安全上下文认证访问 **（不安全的做法，不推荐）**

    首先需要让网关在物理机的局域网内可以被访问
    ```
    # 进入openclaw容器的终端
    docker exec <openclaw容器的ID> /bin/sh
    # 配置允许局域网IP访问网关控制台
    openclaw config set gateway.controlUi.allowedOrigins '["*"]' --json
    ```

    但修改配置文件使局域网内可访问后，登录会显示`control ui requires device identity (use HTTPS or localhost secure context)`。因为控制台 UI 需要设备身份认证，而设备身份的生成机制在浏览器中依赖于安全上下文（即必须是 HTTPS 或者 localhost/127.0.0.1）。由于配置使用了局域网 IP (非 localhost) 且没有配置 HTTPS（即使用了普通的 HTTP），浏览器阻止了设备身份的生成，从而被 OpenClaw 的安全策略拒绝访问。

    因此需要继续修改配置，让openclaw容器允许非安全上下文认证
    ```
    openclaw config set gateway.controlUi.dangerouslyDisableDeviceAuth true --json
    # 重启容器
    docker restart <openclaw容器的ID>
    ```

    **方法二：** 使用Tailscale提供https访问。tailscale是一个组网程序，登录同一个账号的设备可以通过程序生成的专属https链接进行访问，不在设备群的其他设备无法通过该链接访问。
    ```
    # 先确保tailscale已安装并登录
    tailscale status
    # 若没安装，则执行官方安装脚本进行安装（需要sudo权限账户操作）
    curl -fsSL https://tailscale.com/install.sh | sh
    sudo tailscale up
    # 让tailscale监听网关仪表盘，这里会生成一个专属连接
    sudo tailscale serve --bg http://127.0.0.1:18789
    Available within your tailnet:
    #可以得到以下输出（仅供参考）
        https://fds-z790-ud.tail73fbe3.ts.net/
        |-- proxy http://127.0.0.1:18789

        Serve started and running in the background.
        To disable the proxy, run: tailscale serve --https=443 off
    # 重启容器
    docker restart <openclaw容器的ID>
    ```
    后续使用`https://fds-z790-ud.tail73fbe3.ts.net`链接即可访问网关仪表盘。

    在初次访问时，会出现pairing required，这时需要在宿主机中批准该设备的访问。
    ```
    # 列出请求接入的设备
    docker compose run --rm openclaw-cli devices list
    ```
    找到状态为pending的记录，复制它的REQUEST ID。
    ```
    # 批准该请求
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```
    只要不清除浏览器缓存或更换浏览器，可以一直访问无需重新批准。

4. 部署多模态大模型时需要留意是否开启了图像输入
    ```
    # 适合当前账户没有sudo权限时使用
    docker compose run --rm --entrypoint /bin/sh openclaw-cli -c \
    "sed -i '/\"id\": \"<模型的名称>\"/,/\"input\": \[/ {
    /\"text\"/!b
    s/\"text\"/\"text\", \"image\"/
    }' /home/node/.openclaw/openclaw.json"
    # 重启网关
    docker compose restart openclaw-gateway
    ```
5. 资料传输问题

    容器的工作路径一般用户无法访问，因此最好挂载一个公共空间给容器，然后要求龙虾将输出的文件保存在此
    ```
    # 创建文件夹
    mkdir -p /home/openclaw/shared

    # 将该文件夹设为最高权限（所有人可读、可写、可执行）
    chmod 777 /home/openclaw/shared
    # 赋予其他用户“穿透”主目录的权限
    chmod o+x /home/openclaw
    ```

6. approve问题

    一直要approve比较麻烦，可以通过关闭权限审核的方式省事，考虑到运行在docker内相对独立安全，可以关闭权限。（风险高，不推荐）
    ```
    docker compose run --rm --entrypoint /bin/sh openclaw-cli -c "openclaw config set tools.exec.security full && openclaw config set tools.exec.ask off"

    docker compose restart openclaw-gateway
    ```
7. 安装python问题

    在使用的过程中发现龙虾没有调用用户的conda环境（这是好事，不要随便乱动用户的文件），因此决定给龙虾单独装一个miniconda。在问题5处已经设置了一个共享文件夹，因此将conda安装在那里是最优选择。
    ```
    # 下载并静默安装miniconda
    docker compose run --rm --entrypoint /bin/sh openclaw-cli -c "wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O /tmp/miniconda.sh && sh /tmp/miniconda.sh -b -p /home/node/shared/miniconda && mv /tmp/miniconda.sh /home/node/shared/"
    # 为容器挂载miniconda
    docker compose run --rm --entrypoint /bin/sh openclaw-cli -c "echo 'export PATH=\"/home/node/shared/miniconda/bin:\$PATH\"' >> ~/.bashrc && /home/node/shared/miniconda/bin/conda init bash"
    ```    
8. 修改openclaw配置相关

    由于是docker容器部署，普通账号不方便直接修改~/.openclaw/openclaw.json文件。较为简单的方法是切换到一个有sudo权限的账号进行操作。


## 设置聊天平台
原生openclaw支持的平台中，中国内地较为方便的应用为QQ、飞书和Teams，考虑到方便性，优先使用QQ bot，依照[官方教程](https://docs.openclaw.ai/channels/qqbot)

```
# 1. 安装 QQBot 插件
docker compose run --rm openclaw-cli plugins install @openclaw/qqbot

# 2. 绑定你的机器人凭证（这里不要加尖括号，换成你自己的真实值）
docker compose run --rm openclaw-cli config set channels.qqbot.appId "你的AppID"
docker compose run --rm openclaw-cli config set channels.qqbot.clientSecret "你的ClientSecret"

# 3. 启用通道
docker compose run --rm openclaw-cli config set channels.qqbot.enabled true

# 4. 重启 Gateway 使服务生效
docker compose restart openclaw-gateway
```

## 多agent部署
默认只有单个agent，不利于多任务。最好每种使用场景单独设置一个agent，并绑定到不同的QQ bot，方式上下文混乱。

```
# 创建agent及其工作区
docker compose run --rm openclaw-cli agents add coding
```
在创建的过程中可以根据提示一步步完成模型配置和qq bot的配置。
完成后可以通过`docker compose run --rm openclaw-cli agents list --bindings`命令查看绑定情况
