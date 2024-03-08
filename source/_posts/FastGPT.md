---
title: 基于本地大模型的 FastGPT 框架部署
date: 2024-03-08 23:27:35
categories:
- AI
- Sites
tags:
- FastGPT
- Docker Compose
---
# 前言
[FastGPT](https://fastgpt.in) 是一款强大的 LLM + RAG 解决方案，本文记录了基于本地大模型搭建 FastGPT 框架的过程。

<!--more -->

# 整体框架说明

<center>
    <img src="/img/FastGPT/sealos-fastgpt.png" width="850">
</center>

- 整体框架由 4 部分组成，分别为数据库、FastGPT、OneAPI、大模型

## FastGPT

- 框架本体，默认使用 OpenAI 的大模型接口

### 开发组件依赖

- Docker
- Node.js v18.x
- pnmp 版本 8.x.x

## 数据库

- MongoDB 用于存储 FastGPT 框架运行所需的数据
- pgvector 用于存储知识库向量
    
    
    | 环境 | 最低配置（单节点） | 推荐配置 |
    | --- | --- | --- |
    | 测试 | 2c2g | 2c4g |
    | 100w 组向量 | 4c8g 50GB | 4c16g 50GB |
    | 500w 组向量 | 8c32g 200GB | 16c64g 200GB |

## OneAPI

- OneAPI 是大模型调用接口框架，负责对接大模型调用接口，提供权限控制和收费统计等功能

## 本地大模型

- 提供的本地大模型接口需要符合 OpenAI 的接口规范
- 综合显存需求
    - 最少：32GB+
    - 推荐：48GB+

### 对话大模型

- Baichuan 2
- ChatGLM 2/3

### Embedding 模型

- m3e-large

### ReRank 模型

- bge-reranker-large

# 基于 Docker Compose 的快速部署

## 部署本地模型 + OneAPI

- 部署实例中使用的本地模型组合为 Baichuan2-7B-Chat + M3E-large + BGE-ReRanker-base
- 硬件资源和前置需求见下方分布式部署相关章节
- 在本地创建文件夹并下载相关文件
    
    ```bash
    # 创建文件夹
    mkdir llm
    cd llm
    
    # 下载 docker-compose.yml 文件
    curl -O https://raw.githubusercontent.com/Coldwave96/FastGPT-Deploy-Utilities/main/LLM-DockerCompose/docker-compose.yml
    ```
    
- 根据`docker-compose.yml`文件中的提示修改对应的信息
- 通过以下命令控制相关容器
    
    ```bash
    # 启动
    docker compose up -d
    
    # 停止
    docker compose down
    ```
    

## 配置 OneAPI

- 登陆`http://<gpu_server_ip>:8000`，初始账号密码`root/123456` ，登陆成功后及时修改默认密码
- 点击`令牌`→`添加新的令牌`，输入名称，内部使用可设置`永不过期 + 设置无限额度`
- 回到`令牌`，点击`复制`即可获取`Token`
- 点击`渠道`→`添加新的渠道`
    - 添加 Baichuan2-7B-Chat
        - 类型：`OpenAI`
        - 名称：`Baichuan2-7B-Chat`（随意）
        - 模型：`Baichuan2-7B-Chat`（随意，FastGPT 配置文件中与之对应即可）
        - 密钥：本地大模型接口的`SK-KEY` 值
        - 代理：`http://<gpu_server_ip>:8001`
    - 添加 M3E-large
        - 类型：`自定义渠道`
        - Base URL：`http://<gpu_server_ip>:8002`
        - 名称：`M3E-large`（随意）
        - 模型：`M3E-large`（随意，FastGPT 配置文件中与之对应即可）
        - 密钥：本地模型接口的`SK-KEY`值

## 部署 FastGPT

- 在本地创建文件夹并下载相关文件
    
    ```bash
    # 创建文件夹
    mkdir fastgpt
    cd fastgpt
    
    # 下载相关文件
    curl -O https://raw.githubusercontent.com/Coldwave96/FastGPT-Deploy-Utilities/main/FastGPT/docker-compose.yml
    curl -O https://raw.githubusercontent.com/Coldwave96/FastGPT-Deploy-Utilities/main/FastGPT/config.json
    ```
    
    - 注意: docker-compose.yml 配置文件中 Mongo 为 5.x，部分服务器不支持，需手动更改其镜像版本为 4.4.24
- 修改 docker-compose.yml 中的 OPENAI_BASE_URL（API 接口的地址，需要加/v1）和CHAT_API_KEY（API 接口的凭证)
- 使用 OneAPI 的话，OPENAI_BASE_URL=OneAPI访问地址/v1；CHAT_API_KEY=令牌
- 修改 config.json 中的本地问答大模型、Embedding 模型以及 ReRank 模型的相关信息
- 在 docker-compose.yml 同级目录下执行
    
    ```bash
    # 进入项目目录
    cd 项目目录
    # 创建 mongo 密钥
    openssl rand -base64 756 > ./mongodb.key
    # 600不行可以用chmod 999
    chmod 600 ./mongodb.key
    chown 999:root ./mongodb.key
    # 启动容器
    docker compose pull
    docker compose up -d
    ```
    
- 初始化 Mongo 副本集(4.6.8以前可忽略)
    - Mongo 数据库需要修改副本集的host，从原来的mongo:27017修改为ip:27017。
    
    ```bash
    # 查看 mongo 容器是否正常运行
    docker ps
    # 进入容器
    docker exec -it mongo bash
    
    # 连接数据库
    mongo -u myname -p mypassword --authenticationDatabase admin
    
    # 初始化副本集。如果需要外网访问，mongo:27017 可以改成 ip:27017。但是需要同时修改 FastGPT 连接的参数（MONGODB_URI=mongodb://myname:mypassword@mongo:27017/fastgpt?authSource=admin => MONGODB_URI=mongodb://myname:mypassword@ip:27017/fastgpt?authSource=admin）
    rs.initiate({
      _id: "rs0",
      members: [
        { _id: 0, host: "mongo:27017" }
      ]
    })
    # 检查状态。如果提示 rs0 状态，则代表运行成功
    rs.status()
    ```
    

## 访问 FastGPT

- 目前可以通过`http://<ip>:8080`直接访问(注意防火墙)。登录用户名为`root`，密码为`docker-compose.yml`环境变量里设置的`DEFAULT_ROOT_PSW`
- 如果需要域名访问，请自行安装并配置 Nginx

# 分布式部署

## 部署对话大模型

- 选择以下其中一种大模型部署即可

### Baichuan2-13B-Chat

- 推荐配置
    
    
    | 类型 | 内存 | 显存 | 硬盘空间 | 启动命令 |
    | --- | --- | --- | --- | --- |
    | fp16 | ≥ 32GB | ≥ 28GB | ≥ 50GB | python openai_api.py |
    | int8 | ≥ 32GB | ≥ 17GB | ≥ 50GB | python openai_api.py（设置环境变量 QUANTIZE_BIT=8） |
    | int4 | ≥ 32GB | ≥ 9GB | ≥ 50GB | python openai_api.py（设置环境变量 QUANTIZE_BIT=4） |
- 部署环境要求
    - Python 3.10
    - NVIDIA 驱动 + CUDA 等套件
- 源码部署
    - 将 Baichuan2-13B-Chat 模型文件下载到本地
        
        [百川2-13B-对话模型](https://modelscope.cn/models/baichuan-inc/Baichuan2-13B-Chat)
        
    - 下载相关文件
        
        ```bash
        curl -O https://raw.githubusercontent.com/Coldwave96/FastGPT-Deploy-Utilities/main/LLM-Chat/Baichuan2-13B-Chat/openai_api.py
        curl -O https://raw.githubusercontent.com/Coldwave96/FastGPT-Deploy-Utilities/main/LLM-Chat/Baichuan2-13B-Chat/requirements.txt
        ```
        
    - 安装依赖
        
        ```bash
        pip install -r requirements.txt
        ```
        
    - 设置环境变量`SK_KEY`，这是大模型调用接口认证 token，防止接口盗用
    - 修改`openai_api.py`文件中模型名称`baichuan-inc/Baichuan2-13B-Chat`为本地模型所在文件夹
    - 运行启动命令`python openai_api.py`

### Baichuan2-7B-Chat

- 推荐配置
    
    
    | 类型 | 内存 | 显存 | 硬盘空间 | 启动命令 |
    | --- | --- | --- | --- | --- |
    | fp16 | ≥ 16GB | ≥ 16GB | ≥ 25GB | python openai_api.py |
    | int8 | ≥ 16GB | ≥ 9GB | ≥ 25GB | python openai_api.py（设置环境变量 QUANTIZE_BIT=8） |
    | int4 | ≥ 16GB | ≥ 6GB | ≥ 25GB | python openai_api.py（设置环境变量 QUANTIZE_BIT=4） |
- 部署环境要求
    - Python 3.10
    - NVIDIA 驱动 + CUDA 等套件
- 源码部署
    - 将 Baichuan2-7B-Chat 模型文件下载到本地
        
        [百川2-7B-对话模型](https://modelscope.cn/models/baichuan-inc/Baichuan2-7B-Chat)
        
    - 下载相关文件
        
        ```bash
        curl -O https://raw.githubusercontent.com/Coldwave96/FastGPT-Deploy-Utilities/main/LLM-Chat/Baichuan2-7B-Chat/openai_api.py
        curl -O https://raw.githubusercontent.com/Coldwave96/FastGPT-Deploy-Utilities/main/LLM-Chat/Baichuan2-7B-Chat/requirements.txt
        ```
        
    - 安装依赖
        
        ```bash
        pip install -r requirements.txt
        ```
        
    - 设置环境变量`SK_KEY`，这是大模型调用接口认证 token，防止接口盗用
    - 修改`openai_api.py`文件中模型名称`baichuan-inc/Baichuan2-7B-Chat`为本地模型所在文件夹
    - 运行启动命令`python openai_api.py`

### ChatGLM2-6B

- 推荐配置
    
    
    | 类型 | 内存 | 显存 | 硬盘空间 | 启动命令 |
    | --- | --- | --- | --- | --- |
    | fp16 | ≥ 16GB | ≥ 16GB | ≥ 25GB | python openai_api.py |
    | int8 | ≥ 16GB | ≥ 9GB | ≥ 25GB | python openai_api.py（设置环境变量 QUANTIZE_BIT=8） |
    | int4 | ≥ 16GB | ≥ 6GB | ≥ 25GB | python openai_api.py（设置环境变量 QUANTIZE_BIT=4） |
- 部署环境要求
    - Python 3.10
    - NVIDIA 驱动 + CUDA 等套件
- 源码部署
    - 将 ChatGLM2-6B 模型文件下载到本地
        
        [chatglm2-6b](https://modelscope.cn/models/ZhipuAI/chatglm2-6b/)
        
    - 下载相关文件
        
        ```bash
        curl -O https://raw.githubusercontent.com/Coldwave96/FastGPT-Deploy-Utilities/main/LLM-Chat/ChatGLM2-6B/openai_api.py
        curl -O https://raw.githubusercontent.com/Coldwave96/FastGPT-Deploy-Utilities/main/LLM-Chat/ChatGLM2-6B/requirements.txt
        ```
        
    - 安装依赖
        
        ```bash
        pip install -r requirements.txt
        ```
        
    - 设置环境变量`SK_KEY`，这是大模型调用接口认证 token，防止接口盗用
    - 修改`openai_api.py`文件中模型名称`THUDM/chatglm2-6b`为本地模型所在文件夹
    - 运行启动命令`python openai_api.py`

## 部署 Embedding 模型

### M3E-large

- 推荐配置
    
    
    | 内存 | 显存 | 硬盘空间 | 启动命令 |
    | --- | --- | --- | --- |
    | ≥ 8GB | ≥ 6GB | ≥ 10GB | python openai_api.py |
- 部署环境要求
    - Python 3.8
    - NVIDIA 驱动 + CUDA 等套件
- 源码部署
    - 将 M3E-large 模型文件下载到本地
        
        [M3E-large](https://modelscope.cn/models/Jerry0/M3E-large/)
        
    - 下载相关文件
        
        ```bash
        curl -O https://raw.githubusercontent.com/Coldwave96/FastGPT-Deploy-Utilities/main/LLM-Embedding/M3E-large/openai_api.py
        curl -O https://raw.githubusercontent.com/Coldwave96/FastGPT-Deploy-Utilities/main/LLM-Embedding/M3E-large/requirements.txt
        ```
        
    - 安装依赖
        
        ```bash
        pip install -r requirements.txt
        ```
        
    - 设置环境变量`SK_KEY`，这是大模型调用接口认证 token，防止接口盗用
    - 修改`openai_api.py`文件中模型名称`moka-ai/m3e-large`为本地模型所在文件夹
    - 运行启动命令`python openai_api.py`

## 部署 ReRank 重排模型

### BGE-ReRanker-Base

- 推荐配置
    
    
    | 类型 | 内存 | 显存 | 硬盘空间 | 启动命令 |
    | --- | --- | --- | --- | --- |
    | base | ≥ 4GB | ≥ 3GB | ≥ 8GB | python api.py |
- 部署环境要求
    - Python 3.10
    - NVIDIA 驱动 + CUDA 等套件
- 源码部署
    - 将 BGE-ReRanker-base 模型下载到本地
        
        [BAAI/bge-reranker-base · Hugging Face](https://huggingface.co/BAAI/bge-reranker-base)
        
    - 下载相关文件（与存放模型的文件夹在同一级）
        
        ```bash
        curl -O https://raw.githubusercontent.com/Coldwave96/FastGPT-Deploy-Utilities/main/LLM-ReRanker/BGE-ReRanker-base/api.py
        curl -O https://raw.githubusercontent.com/Coldwave96/FastGPT-Deploy-Utilities/main/LLM-ReRanker/BGE-ReRanker-base/requirements.txt
        ```
        
    - 安装依赖
        
        ```bash
        pip install -r requirements.txt
        ```
        
    - 添加环境变量`export ACCESS_TOKEN=XXXXXX`配置 token，这里的 token 只是加一层验证，防止接口被人盗用，默认值为`ACCESS_TOKEN`
    - 修改`api.py`文件中`bge-reranker-large`为存储本地模型文件夹名称
    - 运行启动命令`python api.py`

## 部署 OneAPI

### Docker 部署

```python
# 使用 SQLite 的部署命令：
docker run --name one-api -d --restart always -p 3000:3000 -e TZ=Asia/Shanghai -v /home/ubuntu/data/one-api:/data justsong/one-api

# 使用 MySQL 的部署命令，在上面的基础上添加 `-e SQL_DSN="root:123456@tcp(localhost:3306)/oneapi"`，请自行修改数据库连接参数，不清楚如何修改请参见下面环境变量一节。
# 例如：
docker run --name one-api -d --restart always -p 3000:3000 -e SQL_DSN="root:123456@tcp(localhost:3306)/oneapi" -e TZ=Asia/Shanghai -v /home/ubuntu/data/one-api:/data justsong/one-api

```

- 其中，-p 3000:3000 中的第一个 3000 是宿主机的端口，可以根据需要进行修改
- 数据和日志将会保存在宿主机的`/home/ubuntu/data/one-api`目录，请确保该目录存在且具有写入权限，或者更改为合适的目录
- 如果启动失败，请添加`-privileged=true`
- 访问`http://<ip>:3000/`并登录。初始账号用户名为`root`，密码为`123456`

### 创建令牌

- 可设置永不过期，无限额度

### 接入本地问答大模型

- 类型：OpenAI
- 名称：随便写
- 模型：自定义模型名称
- 代理：本地大模型开放的 OpenAI 格式的 API 接口地址

### 接入本地 Embedding 模型

- 类型：自定义渠道
- Base URL：Embedding 模型开放的 OpenAI 格式的 API 接口地址
- 名称：随便写
- 模型：自定义模型名称
- 密钥：开放接口定义的密钥

## 部署 FastGPT

### Docker Compose 快速部署

- 依次执行下面命令，创建 FastGPT 文件并拉取docker-compose.yml和config.json，执行完后目录下会有 2 个文件
    
    ```bash
    mkdir fastgpt
    cd fastgpt
    curl -O https://raw.githubusercontent.com/Coldwave96/FastGPT-Deploy-Utilities/main/FastGPT/docker-compose.yml
    curl -O https://raw.githubusercontent.com/Coldwave96/FastGPT-Deploy-Utilities/main/FastGPT/config.json
    ```
    
    - 注意: docker-compose.yml 配置文件中 Mongo 为 5.x，部分服务器不支持，需手动更改其镜像版本为 4.4.24
- 修改 docker-compose.yml 中的 OPENAI_BASE_URL（API 接口的地址，需要加/v1）和CHAT_API_KEY（API 接口的凭证)
- 使用 OneAPI 的话，OPENAI_BASE_URL=OneAPI访问地址/v1；CHAT_API_KEY=令牌
- 修改 config.json 中的本地问答大模型、Embedding 模型以及 ReRank 模型的相关信息
- 在 docker-compose.yml 同级目录下执行
    
    ```bash
    # 进入项目目录
    cd 项目目录
    # 创建 mongo 密钥
    openssl rand -base64 756 > ./mongodb.key
    # 600不行可以用chmod 999
    chmod 600 ./mongodb.key
    chown 999:root ./mongodb.key
    # 启动容器
    docker compose pull
    docker compose up -d
    ```
    
- 初始化 Mongo 副本集(4.6.8以前可忽略)
    - Mongo 数据库需要修改副本集的host，从原来的mongo:27017修改为ip:27017。
    
    ```bash
    # 查看 mongo 容器是否正常运行
    docker ps
    # 进入容器
    docker exec -it mongo bash
    
    # 连接数据库
    mongo -u myname -p mypassword --authenticationDatabase admin
    
    # 初始化副本集。如果需要外网访问，mongo:27017 可以改成 ip:27017。但是需要同时修改 FastGPT 连接的参数（MONGODB_URI=mongodb://myname:mypassword@mongo:27017/fastgpt?authSource=admin => MONGODB_URI=mongodb://myname:mypassword@ip:27017/fastgpt?authSource=admin）
    rs.initiate({
      _id: "rs0",
      members: [
        { _id: 0, host: "mongo:27017" }
      ]
    })
    # 检查状态。如果提示 rs0 状态，则代表运行成功
    rs.status()
    ```
    

### 访问 FastGPT

- 目前可以通过 ip:8080 直接访问(注意防火墙)。登录用户名为 root，密码为 docker-compose.yml 环境变量里设置的 DEFAULT_ROOT_PSW
- 如果需要域名访问，请自行安装并配置 Nginx
