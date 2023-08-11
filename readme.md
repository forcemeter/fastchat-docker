# 使用 `FastChat` 运行 `Baichuan-13B-Chat` 和 `Qwen-7B-Chat`

为了对比`Baichuan-13B-Chat` 和 `Qwen-7B-Chat`的效果，计划采用`FastChat`的`Chatbot Arena`实现。

官方的 `dockerfile` 只包含 `controller` 和 `worker`，所以我单独做了一套，包含 `controller`, `worker`, `api`, `webui`。

目前还未确认能支持 `vllm`，努力中。

## step 1, 官方源码下载

docker源码地址

> https://github.com/lm-sys/FastChat

文档地址

> https://www.4wei.cn/archives/1003130

## step 1, FastChat 的源码安装，构建镜像

> 官方源码地址，可通过 git 或者手工下载

`https://github.com/lm-sys/FastChat`

虽然可以通过 `pip3 install fschat` 来快速安装，但为了使用最新版本，且在 docker 中使用，需要手动构建镜像。

## step 2, 镜像构建

```shell

# step 1 中实现
# git clone https://github.com/lm-sys/FastChat
cd FastChat
git clone github.com/forcemeter/fastchat-docker 4wei
docker build . -f 4wei/dockerfile -t fastchat:latest

# 如果是在中国，可以使用中国源
#docker build . -f 4wei/dockerfile-china -t fastchat:latest
```

## step 3, 检查显卡配置

首先检查 GPU 是否满足需求，能同时运行这两个模型，至少需要 `60G` 显存，我使用了 4块 * 24G的 `P40`，所以在使用 docker 的时候，要进行分配。

```text
(FastChat) [root@lm-sys-01 FastChat]# nvidia-smi
Fri Aug 11 23:31:24 2023
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 515.105.01   Driver Version: 515.105.01   CUDA Version: 11.7     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla P40           Off  | 00000000:00:0C.0 Off |                    0 |
| N/A   22C    P8     9W / 250W |      0MiB / 23040MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
|   1  Tesla P40           Off  | 00000000:00:0D.0 Off |                    0 |
| N/A   20C    P8     9W / 250W |      0MiB / 23040MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
|   2  Tesla P40           Off  | 00000000:00:0E.0 Off |                    0 |
| N/A   22C    P8     9W / 250W |      0MiB / 23040MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
|   3  Tesla P40           Off  | 00000000:00:0F.0 Off |                    0 |
| N/A   23C    P8    12W / 250W |      0MiB / 23040MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

## step 4, GPU 分配及集群设置

在 `dockercompose.yml` 分别指定不同的 GPU 和显存上限，千问我分配了24G，百川分配了双卡共48G。

```
  #百川
  fastchat-worker-baichuan:
    ...
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['2', '3']
              capabilities: [gpu]
    entrypoint: ["python3.9", "-m", "fastchat.serve.model_worker", ..., "--device=cuda", "--num-gpus=2", "--max-gpu-memory=24GB"]
```

`fastchat` 支持分布式 `worker` 架构，我这里是单机多卡，唯一的区别是设置 `worker` 的控制器参数。

```
--controller-address=http://fastchat-controller:21001
```

如果你离线下载了所有模型，可以在 `docker-compose.yml` 中指定挂载路径，否则会自动下载。

模型的下载命令，目前国内可用:

```shell
git clone git@hf.co:Qwen/Qwen-7B-Chat
git clone git@hf.co:baichuan-inc/Baichuan-13B-Chat
```

## 启动服务

```shell
# 前台启动
docker compose up

# 后台启动
docker compose up -d

# 查看进程
docker compose ps

# 进入容器
docker compose exec -it fastchat-controller bash
```

然后访问 `http://127.0.0.1:7860`