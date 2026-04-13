# 以docker部署comfyui记录

## 前言

comfyui是一个基于节点工作流的stable diffusion图像界面，可以通过可视化的方式组合复杂的图像生成流程。

为了方便迁移项目、修改端口映射等功能，同时为了测试在一个相对干净的容器中自行搭建自己的项目容器，本次comfyui将使用docker进行部署。

## 部署过程

1. 由于官方没有给对应的docker镜像，需要手动从头搭建。因此先建立一个最基础的容器。由于会用到Nvidia显卡加速，因此将系统中的cuda挂载进去。此处宿主机安装的是cuda13.1，宿主机系统为ubuntu22.04

```
docker run -it \
  --name comfyui-docker \
  --gpus all \
  -p 8188:8188 \
  -v $(pwd)/comfyui_workspace:/workspace \
  -v /usr/local/cuda-13.1:/usr/local/cuda-13.1:ro \
  -v /usr/local/cuda:/usr/local/cuda:ro \
  -e PATH="/usr/local/cuda-13.1/bin:/usr/local/cuda/bin:$PATH" \
  -e LD_LIBRARY_PATH="/usr/local/cuda-13.1/lib64:/usr/local/cuda/lib64:$LD_LIBRARY_PATH" \
  -e CUDA_HOME="/usr/local/cuda-13.1" \
  ubuntu:22.04 \
  /bin/bash
```

上述命令最后2行的含义：使用ubuntu22.04镜像，并在容器启动后立刻打开一个终端。

进入容器终端后可以验证一下显卡驱动是否挂载成功

Note：对于没有显式挂载的其他系统目录（如/bin、/etc、/usr等），是基于基础镜像生成，挂载在一个由docker管理等虚拟文件管理系统中，与宿主机的系统目录互不干扰。因此一般在创建容器时不要挂载系统目录。在上述创建容器命令中，虽然挂载了系统的cuda驱动目录，但在结尾处添加了`:ro`，代表以只读的模式挂载，无需担心容器修改系统文件。


```
root@8769e233acd9:/# nvcc -V

nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2025 NVIDIA Corporation
Built on Fri_Nov__7_07:23:37_PM_PST_2025
Cuda compilation tools, release 13.1, V13.1.80
Build cuda_13.1.r13.1/compiler.36836380_0
```

2. 安装python，部署comfyui环境。由于是在容器内部使用，无需使用anaconda管理多个虚拟环境，直接安装基础的python发行版本即可。
comfyui推荐的python版本为3.13，ubuntu自带的python版本为3.10，需要额外安装。另外ubuntu安装源中暂时不包括python3.13，且基础ubuntu镜像不包含安装源管理工具，需要额外安装。
```
# 更新软件列表并安装管理工具和其他的一些必要工具
apt-get update
apt-get install -y software-properties-common git

# 添加deadsnakes PPA源
add-apt-repository -y ppa:deadsnakes/ppa
apt-get update

# 安装python3.13
apt-get install -y python3.13 python3.13-dev python3.13-venv
```

由于通过上述方法安装的python通常没有pip，需要手动下载官方安装脚本安装pip，并把python命令指向python3.13：

```
# 下载并运行 pip 安装脚本
wget https://bootstrap.pypa.io/get-pip.py
python3.13 get-pip.py

# 修改系统的软链接，让默认的 python 命令直接指向 python3.13
ln -sf /usr/bin/python3.13 /usr/bin/python
ln -sf /usr/bin/python3.13 /usr/bin/python3
```

3. 进入容器的工作路径中安装comfyui

```
cd /workspace

git clone https://github.com/Comfy-Org/ComfyUI.git

cd ComfyUI

pip install torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/cu130

pip install -r requirements.txt

pip install -r manager_requirements.txt
```

4. 启动comfyui，检查各项功能是否正常

```
python main.py --enable-manager --listen 0.0.0.0
```

默认端口为8188，在创建容器时可以自行修改映射。

5. 当各项功能正常后，为了迁移项目和打开容器时自动开启comfyui，需要将当前已部署好的容器打包为一个新的镜像并安装到一个新的容器。

```
# 打包现有容器为新的镜像
docker commit comfyui-docker comfyui-docker-v1

# 停止并删除旧容器
docker stop comfyui-docker
docker rm comfyui-docker

# 利用新镜像开启新容器并开启后台启动和开机自启动
docker run -d \
  --name comfyui \
  --restart unless-stopped \
  --gpus all \
  -p 18188:8188 \
  -v $(pwd)/comfyui_workspace:/workspace \
  -v /usr/local/cuda-13.1:/usr/local/cuda-13.1:ro \
  -v /usr/local/cuda:/usr/local/cuda:ro \
  -e PATH="/usr/local/cuda-13.1/bin:/usr/local/cuda/bin:$PATH" \
  -e LD_LIBRARY_PATH="/usr/local/cuda-13.1/lib64:/usr/local/cuda/lib64:$LD_LIBRARY_PATH" \
  -e CUDA_HOME="/usr/local/cuda-13.1" \
  -w /workspace/ComfyUI \
  comfyui-docker-v1 \
  python main.py --enable-manager --listen 0.0.0.0

# 查看后台运行日志
docker logs -f comfyui 
```