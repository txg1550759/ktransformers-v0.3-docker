# ktransformers-v0.3-docker
ktransformers v0.3 docker build and run

Current Hardware Environment Configuration：

目前环境硬件配置：
```
 Xeon CPU, 2 Intel(R) Xeon(R) Gold 5320 CPUs, 2 A800 80G GPUs, 512G DDR4 memory. The model is DeepSeek-R1-GGUF/DeepSeek-R1-Q4_K_M
至强第三代cpu, 2块Intel(R) Xeon(R) Gold 5320, 2块A800 80G，512G DDR4 内存，模型是DeepSeek-R1-GGUF/DeepSeek-R1-Q4_K_M

```
Software Environment Configuration：

软件环境配置:
```
cuda12.6
python3.11
pytorch/pytorch:2.6.0-cuda12.6-cudnn9-devel
flash_attn                2.7.4.post1
GLIBCXX_3.4.32

```

This Dockerfile is built based on a proxy. The prerequisite is that you need to set up an HTTP proxy to access GitHub by yourself. Because the build will fail without a proxy.
If you are abroad and have no network issues accessing GitHub, you can build without a proxy. You can modify my Dockerfile and remove the proxy settings.
Since the Chinese network cannot access GitHub, which will lead to a build failure, a proxy needs to be introduced for downloading when GitHub cannot be accessed. Follow the steps below:

本dockerfile基于代理构建，前提是条件需要自己搭建http代理访问github, 因为不经过代理，构建会失败。
如果你在国外访问githb无网络问题，可以不经过代理，你可以基于我的dockerfile进行改造，去掉代理。
由于中国网络无法访问github会导致构建失败，所以在无法访问github的情况需要引入代理进行下载，执行以下步骤：

#Set Your Proxy

#设置你的proxy

```
export  http_proxy=http://192.168.1.10:10809
export https_proxy=http://192.168.1.10:10809
```

1、Generate the Dependency File third_party

1、生成依赖文件third_party
 ```
mkdir -p build-v0.3
cd build-v0.3/
git clone https://github.com/kvcache-ai/ktransformers.git 
git submodule init 
git submodule update 
```

2、Download the Original v0.3 Installation File

2、下载原v0.3安装文件
```
wget https://github.com/kvcache-ai/ktransformers/releases/download/v0.1.4/ktransformers-0.3.0rc0+cu126torch26fancy-cp311-cp311-linux_x86_64.whl
```
3、Download the Content of the proxy-dockerfile in This Repository, Rename It to Dockerfile, and Build Your Image

3、下载本仓库proxy-dockerfile内容，重命名为Dockerfile， 构建你的镜像
```
docker build  -t   test-ktransformers:v0.3 .
```
4、For CPUs That Do Not Support AMX Instructions, Run the Following Commands

4、普通不支持AMX的CPU指令的请运行
```
docker run --gpus all  -v /data/LLM_Project/DeepSeek-R1-GGUF/DeepSeek-R1-Q4_K_M:/models --name ktransformers -itd test-ktransformers:v0.3
docker exec -it ktransformers /bin/bash
cd  /workspace/ktransformers
python -m ktransformers.local_chat  --gguf_path "/models"  --model_path "/models" --cpu_infer 24 --max_new_tokens 3000  --optimize_rule_path   ./ktransformers/optimize/optimize_rules/DeepSeek-V3-Chat.yaml

flashinfer not found, use triton for linux
using custom modeling_xxx.py.
__AVX512F__
Injecting model as ktransformers.operators.models . KDeepseekV2Model
Injecting model.embed_tokens as default
Injecting model.layers as default
Injecting model.layers.0 as default
Injecting model.layers.0.self_attn as ktransformers.operators.attention . KDeepseekV2Attention
Illegal instruction (core dumped)
```

5、For CPUs That Support AMX Instructions, Run the Following Commands

5、支持AMX cpu指令请运行

#Check if AMX Instructions Are Supported. If amx Is Found, Execute the Following Commands; If Not, It Means the CPU Does Not Support AMX, and You Don't Need to Continue

#检查是否支持AMX指令,如有amx找到则执行, 如没有找到则不支持，不用继续
```
cat /proc/cpuinfo |grep amx
docker run --gpus all  -v /data/LLM_Project/DeepSeek-R1-GGUF/DeepSeek-R1-Q4_K_M:/models --name ktransformers -itd test-ktransformers:v0.3
docker exec -it ktransformers /bin/bash
cd /workspace/ktransformers
```
#Load AMX Instruction Support, Write to the File preload_amx.c, Use vim preload_amx.c

#加载AMX指令支持,写入文件 preload_amx.c ， vim  preload_amx.c
```
//preload_amx.c
#include <stdio.h>
#include <sys/syscall.h>
#include <unistd.h>

#define ARCH_REQ_XCOMP_PERM     0x1023
#define XFEATURE_XTILEDATA      18

__attribute__((constructor)) void init() {
 if (syscall(SYS_arch_prctl, ARCH_REQ_XCOMP_PERM, XFEATURE_XTILEDATA))
 {
    printf("\n Fail to do XFEATURE_XTILEDATA \n\n");
 }
 else
 {
    printf("\n TILE DATA USE SET - OK \n\n");
 }
}
```
#Run with AMX Instruction Support

#加载AMX指令运行
```
root@a9046bcba61e:/workspace/ktransformers# LD_PRELOAD=./preload_amx.so  python -m ktransformers.local_chat  --gguf_path "/models"  --model_path "/models" --cpu_infer 56 --max_new_tokens 20000  --optimize_rule_path   ./ktransformers/optimize/optimize_rules/DeepSeek-V3-Chat-multi-gpu.yaml

 Fail to do XFEATURE_XTILEDATA 

flashinfer not found, use triton for linux
using custom modeling_xxx.py.
__AVX512F__
Injecting model as ktransformers.operators.models . KDeepseekV2Model
Injecting model.embed_tokens as default
Injecting model.layers as default
Injecting model.layers.0 as default
Injecting model.layers.0.self_attn as ktransformers.operators.attention . KDeepseekV2Attention
Illegal instruction (core dumped)

```

