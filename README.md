# ktransformers-v0.3-docker
ktransformers v0.3 docker build and run

本dockerfile基于代理构建，前提是条件需要自己搭建http代理访问github, 因为不经过代理，构建会失败。

由于中国网络无法访问github会导致构建失败，所以在无法访问github的情况需要引入代理进行下载，执行以下步骤：

#设置你的proxy
```
export  http_proxy=http://192.168.1.10:10809
export https_proxy=http://192.168.1.10:10809
```

1、生成依赖文件third_party
 ```
mkdir -p build-v0.3
cd build-v0.3/
git clone https://github.com/kvcache-ai/ktransformers.git 
git submodule init 
git submodule update 
```

2、下载原v0.3安装文件
```
wget https://github.com/kvcache-ai/ktransformers/releases/download/v0.1.4/ktransformers-0.3.0rc0+cu126torch26fancy-cp311-cp311-linux_x86_64.whl
```

3、下载本仓库proxy-dockerfile内容，重命名为Dockerfile， 构建你的镜像
```
docker build  -t   test-ktransformers:v0.3 .
```
4、普通不支持AMX的CPU指令的请运行， 目前我测试的是至强第三代cpu, 2块Intel(R) Xeon(R) Gold 5320, 2块A800 80G，512G内存，报core dumped
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


5、支持AMX cpu指令请运行
#检查是否支持AMX指令,如有amx找到则执行, 如没有找到则不支持，不用继续
```
cat /proc/cpuinfo |grep amx
docker run --gpus all  -v /data/LLM_Project/DeepSeek-R1-GGUF/DeepSeek-R1-Q4_K_M:/models --name ktransformers -itd test-ktransformers:v0.3
docker exec -it ktransformers /bin/bash
cd /workspace/ktransformers
```
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

