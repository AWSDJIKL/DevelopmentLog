# ubuntu装机相关

以下内容包括一台干净的ubuntu系统装机后要做的事情，包括clash的部署。

1. 部署clash+proxychains（境外网络可跳过）。

    [参考教程](https://lai-jx.github.io/2023/11/07/proxy/)

    clash已包含在该仓库中，请自行取用。
    ```
    # 假设clash已在~/路径下
    mkdir Clash

    mv clash ./Clash

    cd ./Clash

    chmod +x clash

    # 下载订阅配置文件，或者通过sftp等途径从其他系统中直接传输过来，建议配置文件名称修改为config.yaml
    wget -O config.yaml [订阅链接]

    # 开启clash，后台会完成各类初始化设置
    ./clash -d .
    ```

    安装proxychains，可以强制单条命令走代理，避免系统代理下其他无关服务浪费流量，更方便监控代理情况。

    ```
    sudo apt update

    sudo apt install proxychains4 vim

    sudo vim /etc/proxychains.conf

    # 在结尾处修改为以下内容
    [ProxyList]
    socks5  127.0.0.1 7890
    ```

    设置clash开机自启动。

    ```
    #创建service文件
    sudo touch /etc/systemd/system/clash.service
    #编辑service文件 
    sudo vim /etc/systemd/system/clash.service 
    # 在编辑页面以下内容

    [Unit] 
    Description=clash daemon  
    [Service] 
    Type=simple 
    User=root 
    ExecStart=~/Clash/clash -d ~/Clash/ 
    Restart=on-failure  
    [Install] 
    WantedBy=multi-user.target


    #注册服务并开启服务
    sudo systemctl enable clash
    sudo systemctl start clash

    #检查服务运行情况
    sudo systemctl status clash 
    ```

    可以检查一下是否成功
    ```
    proxychains4 curl www.google.com

    # 若缺少curl可以先安装
    sudo apt install curl
    ```
    注意proxychains无法作用于ping命令。因为ping命令使用ICMP，位于网络层，而proxychains只能代理TCP/UDP，位于更高的传输层，无法干涉下层的网络层。因此测试代理时建议使用使用curl命令。
2. 安装Nvidia显卡驱动

    以下命令均可在开头添加proxychains4强制走代理，如`proxychains4 sudo apt update`
    下文不在赘述。


    查看当前系统对应的可安装的显卡驱动，选取版本号最高的开源版本（即带有open后缀的），通常更稳定。
    ```
    sudo apt update
    sudo ubuntu-drivers devices
    ```
    此处以590版本为例。

    ```
    sudo apt install nvidia-driver-590-open
    ```

    安装后尽量关闭ubuntu系统内核更新，锁定内核版本。因为重启后ubuntu有概率自动更新系统内核，与显卡驱动版本对不上，出现掉显卡驱动现象。
    ```
    # 确认当前版本号
    uname -r
    # 列出所有内核包
    dpkg -l 'linux-image-*' 'linux-headers-*' | awk '/^ii/ {print $2}'

    # hold所有与当前版本号相同的内核包
    sudo apt-mark hold <对应包名>


    # 建议 hold 元包，防止 meta-package 拉入新内核
    sudo apt-mark hold linux-generic linux-image-generic linux-headers-generic
    ```

    重启系统，检查是否安装成功。
    ```
    sudo reboot

    nvidia-smi
    ```

3. 安装cuda

    安装cuda前首先要安装g++-13和gcc13。
    ```
    # 添加来源，否则大概率搜不到g++-13和gcc13
    sudo add-apt-repository ppa:ubuntu-toolchain-r/test

    # 安装前置软件
    sudo apt update
    sudo apt upgrade
    sudo apt install build-essential

    # 安装
    sudo apt install gcc-13
    sudo apt install g++-13

    # 修改系统默认使用的gcc与g++版本
    sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-13 13

    sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-13 13
    ```

    查看当前cuda最高支持版本，从[cuda历史版本](https://developer.nvidia.com/cuda-toolkit-archive)中选取对应版本。

    ```
    nvidia-smi

    # 此处假设cuda最高支持版本为13.1

    wget https://developer.download.nvidia.com/compute/cuda/13.1.0/local_installers/cuda_13.1.0_590.44.01_linux.run

    sudo sh cuda_13.1.0_590.44.01_linux.run
    ```

    安装完成后，需要将cuda添加到系统路径中。

    ```
    vim ~/.bashrc
    ```
    在结尾处添加以下内容，注意修改为对应的cuda版本，此处以13.1为例。
    ```
    export PATH=/usr/local/cuda/bin:$PATH
    export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
    export PATH=/usr/local/cuda-13.1/bin${PATH:+:${PATH}}
    export LD_LIBRARY_PATH=/usr/local/cuda-13.1/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}
    export CUDA_HOME=/usr/local/cuda-13.1
    ```

    使其立刻生效并验证。
    ```
    source ~/.bashrc
    nvcc -V
    ```
4. 安装cudnn

    最简单的一步，对着官方教程做即可。根据cuda版本，从[cudnn历史版本](https://developer.nvidia.com/cudnn-archive)中选取对应的版本，一般选取发布日期最接近的版本。此处以cudnn9.17.0为例。

    ```
    wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb

    sudo dpkg -i cuda-keyring_1.1-1_all.deb

    sudo apt-get update

    sudo apt-get -y install cudnn9-cuda-13
    ```