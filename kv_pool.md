# KV Cache Pool（Ascend Store）

## 特性介绍（必选）

**内容缺失，需要补充。以下内容基于原文上下文推断，请人工确认：** KV Cache Pool（Ascend Store）是 vLLM-Ascend 提供的跨节点 KV Cache 池化存储特性，通过 `AscendStoreConnector` 将 KV Cache 写入外部存储后端（Mooncake / Memcache / Yuanrong），并配合 PD（Prefill/Decode）分离架构，在多个 vLLM 实例之间共享与复用 KV Cache，避免相同前缀的重复计算。

**内容缺失，需要补充。以下内容基于原文上下文推断，请人工确认：** 主要收益包括：PD 分离部署下 Prefill 与 Decode 节点内存解耦、可独立扩缩容；跨请求/跨节点 KV 复用，减少重复 Prefill 计算开销；支持 SSD Offload 扩展 KV 容量；支持传输 QoS 优先级控制。

**内容缺失，需要补充。以下内容基于原文上下文推断，请人工确认：** 依赖 vLLM main branch 与 vLLM-Ascend main branch，mooncake >= 0.3.11.post1，CANN >= 8.5.0，自上述版本起支持。

**依据原文上下文内容重组，请进行人工校验。**

| 后端 | 核心能力 |
| :--- | :--- |
| Mooncake | Mooncake 是 Kimi（Moonshot AI 提供的 LLM 服务）的 serving 平台。支持 SSD Offload、多租户配额管理，覆盖 A2/A3/Ascend 950 Products 系列。 |
| Memcache | 基于 MemFabric，支持 A3 HCCS 高速互通、分离部署、SSD Cache；`use_layerwise` 逐层 KV 存取仅支持 Memcache 后端（Prefill 节点）。 |
| Yuanrong | 基于 openyuanrong-datasystem，支持 Coordinator/etcd 服务发现、Remote H2D 传输、多节点部署。 |

### 使用场景（必选）

**内容缺失，需要补充。以下内容基于原文上下文推断，请人工确认：**

| 场景 | 区分依据 | 适用条件 | 边界 |
| :--- | :--- | :--- | :--- |
| Mooncake 后端 + PD 分离 | 需要跨节点共享 KV，Prefill/Decode 分节点部署 | A2/A3/Ascend 950 Products，ROCE/HCCS 互联 | 各节点需同步 `PYTHONHASHSEED`；Store/PD 流量分离需 CANN >= 9.1.0 |
| Mooncake 后端 + PD-Mixed | 单节点内 P/D 混合部署，无需启动 proxy | 单节点部署，`kv_role=kv_both` | 请求直接发送到混合部署脚本所在端口 |
| MooncakeStore SSD Offload | 需要将 KV 卸载到 SSD 扩展容量 | `enable_ssd_offload=true`，使用 Embedded Real Client 模式 | 需对齐 1 GB；`ssd_offload_path` 必须为绝对路径；需显式配置 `MOONCAKE_OFFLOAD_TOTAL_SIZE_LIMIT_BYTES` |
| Memcache 后端 | 需要 A3 HCCS 高速互通或逐层存取 | A2/A3/Ascend 950 Products；`use_layerwise` 仅 Prefill 节点 | 分离部署模式目前仅支持 A3 HCCS 场景 |
| Yuanrong 后端 | 需要多节点部署或 Remote H2D 传输 | 已安装 openyuanrong-datasystem，Coordinator 或 etcd 服务发现 | 多节点下 Worker 地址不能使用 `127.0.0.1`/`0.0.0.0`；同一 Worker 不能同时配置两种服务发现后端 |

### 约束与限制（必选）

**内容缺失，需要补充。以下内容基于原文上下文推断，请人工确认：**

| 维度 | 约束与限制 |
| :--- | :--- |
| 硬件 | A2（800I/800T A2）：建议 HDK >= 25.5，ROCE 直传需 `HCCL_INTRA_ROCE_ENABLE=1`。A3（800I/800T A3）：HDK >= 26.0（或 HDK >= 25.5 且 mooncake >= v0.3.11），CANN >= 9.0.0，灵衢计算网络 >= 1.5。Ascend 950 Products（950PR/950DT）：HDK >= 25.6 且 mooncake >= v0.3.11，CANN >= 9.1.0，需额外挂载 `/dev/ummu`、`/dev/uburma`、`/usr/bin/urma_admin`、`/lib/route.conf`、`/etc/hccl_rootinfo.json`。Mooncake wheel 需 glibc >= 2.35。 |
| 部署场景 | 环境需存在 `hccn.conf`，Docker 场景需挂载到容器；Store/PD 流量分离需 CANN >= 9.1.0，面向 A3 与 Ascend 950 Products；Memcache 分离部署模式目前仅支持 A3 HCCS 场景。 |
| 引擎 | 依赖 vLLM main branch 与 vLLM-Ascend main branch。 |
| 模型 | `kv_load_failure_policy=recompute` 暂不支持混合注意力模型（如 DeepSeekV4、Qwen 3.5）；MLA 模型可通过 `consumer_is_to_put` 由 Decode 节点存入 KV 供 Prefill 节点使用。 |
| 特性互斥 | `use_layerwise` 仅支持 Prefill 节点且需 Memcache 后端；Yuanrong Worker 上不能同时配置 Coordinator 与 etcd 两种服务发现后端；`P2P_TRANSFER` 或 FabricMem 模式下无论 `enable_dev_mem_pregister` 取值为何，都会跳过客户端设备内存预注册。 |
| 软件依赖 | CANN >= 8.5.0；mooncake >= 0.3.11.post1（非 default 租户需 >= 0.3.12）；Memcache 需 `memfabric-hybrid` 与 `memcache-hybrid`（SSD Cache 需 `memcache_hybrid >= 1.2.0`）；Yuanrong 需 `openyuanrong-datasystem`。 |
| 其他限制 | 所有节点必须同步 `PYTHONHASHSEED`；`ssd_offload_path` 必须为绝对路径，拒绝相对路径、符号链接及含 `..` 的路径；`ubsio.disk.path` 指定的设备必须专用且无挂载点；多节点部署下 Yuanrong Worker 地址不能使用 `127.0.0.1`/`0.0.0.0`；`tenant_id` 不是认证机制。 |
| 方案约束 | A3 + `ASCEND_ENABLE_USE_FABRIC_MEM=1` 下 fabric mem 分配必须为 1 GB 整数倍；`MOONCAKE_OFFLOAD_TOTAL_SIZE_LIMIT_BYTES` 必须显式设置为与真实磁盘容量匹配；Memcache 分离部署时 `ock.mmc.local_service.max.dram.size` 需容纳所有 LocalService 进程的最大 `dram.size`。 |

## 特性使用（必选）

### 环境准备（可选）

**依据原文上下文内容重组，请进行人工校验。**

1. 检查并确认 `hccn.conf` 文件存在于环境中，若使用 Docker 需挂载到容器：
   ```bash
   cat /etc/hccn.conf
   ```
2. 对于 Ascend 950 Products，额外挂载 `/dev/ummu`、`/dev/uburma`、`/usr/bin/urma_admin`、`/lib/route.conf`、`/etc/hccl_rootinfo.json`。
3. 在所有节点同步 `PYTHONHASHSEED` 环境变量：
   ```bash
   export PYTHONHASHSEED=0
   ```
4. 根据所选后端安装软件（详见各场景中的安装步骤）。

> 说明：Memcache 后端的专用前置检查（内存扫描、Ascend 950 Products 关闭签名验证与容器挂载、SSD 磁盘状态检查）属于场景内步骤，见「场景二：Memcache 后端 步骤 1」。

### 使用样例（必选）

**依据原文上下文内容重组，请进行人工校验。**

#### 场景一：Mooncake 后端

##### 步骤 1：软件安装

检查 Mooncake 的 Wheel 包依赖：

```shell
ldd --version
```

glibc 版本需 >= 2.35。

安装 Mooncake：

```shell
python3 -m pip install mooncake-transfer-engine-npu==0.3.11.post1 --extra-index-url https://mirrors.aliyun.com/pypi/web/simple
```

Mooncake `0.3.11.post1` 在 `tenant_id` 省略或为 `default` 时仍受支持。非 default 租户需使用 mooncake `0.3.12` 或更新版本。

##### 步骤 2：配置 mooncake.json 并启动 mooncake_master

配置 `mooncake.json`，将环境变量 `MOONCAKE_CONFIG_PATH` 指向其完整路径：

```json
{
    "metadata_server": "P2PHANDSHAKE",
    "protocol": "ascend",
    "device_name": "",
    "master_server_address": "xx.xx.xx.xx:50088",
    "global_segment_size": "1GB",
    "preferred_segment": false,
    "prefer_alloc_in_same_node": true,
    "enable_ssd_offload": false,
    "ssd_offload_path": "/nvme/mooncake_offload",
    "tenant_id": "default"
}
```

启动 `mooncake_master`（仅需在单个节点上运行）：

```shell
mooncake_master --port 50088 --eviction_high_watermark_ratio 0.9 --eviction_ratio 0.1 --default_kv_lease_ttl 11000 --enable_offload=false --client_ttl=120
```

如需启用严格多租户隔离，使用如下命令启动：

```shell
mooncake_master \
    --port 50088 \
    --enable_multi_tenants=true \
    --tenant_quota_connector_type=file \
    --tenant_quota_connector_uri=/etc/mooncake/tenant_quotas.yaml
```

租户配额文件示例（`/etc/mooncake/tenant_quotas.yaml`）：

```yaml
version: 1

tenants:
  - name: tenant-a
    quota: 200GB
  - name: tenant-b
    quota: 200GB
  - name: default
    quota: 100GB
```

##### 步骤 3：PD 分离场景

**run_prefill.sh / run_decode.sh：**

```shell
#!/bin/bash

# prefill / decode
ROLE="prefill"
# A2 (800I/800T A2) or A3 (800I/800T A3) or A5 (950PR/950DT)
HARDWARE_SERIES="A2"
# Link type: ROCE or HCCS in A3 series.
LINK_TYPE="ROCE"
LOCAL_IP="xx.xx.xx.xx"
NIC_NAME="xxxxxx"

MODEL_PATH="xxxxxxx/Qwen3-32B"
SERVED_MODEL_NAME="qwen3"
DATA_PARALLEL_SIZE=1
TENSOR_PARALLEL_SIZE=8
export ASCEND_RT_VISIBLE_DEVICES=0,1,2,3,4,5,6,7

# parameters required for kv pool and mooncake
export PYTHONHASHSEED=0
export MOONCAKE_CONFIG_PATH="/xxxxxx/mooncake.json"
export LD_LIBRARY_PATH=/usr/local/Ascend/ascend-toolkit/latest/python/site-packages/mooncake:$LD_LIBRARY_PATH

if [ "$ROLE" == "prefill" ]; then
    KV_ROLE="kv_producer"
    KV_PORT="20001"
    LOOKUP_RPC_PORT="0"
    API_PORT="8100"
else
    KV_ROLE="kv_consumer"
    KV_PORT="20002"
    LOOKUP_RPC_PORT="1"
    API_PORT="8200"
fi

echo "Starting vLLM on Series: $HARDWARE_SERIES, Role: $ROLE"

rm -rf /root/ascend/log/*
rm -rf ./connector.log

# 详细参数说明见「配置参数」章节
if [ "$HARDWARE_SERIES" == "A2" ] || { [ "$HARDWARE_SERIES" == "A3" ] && [ "$LINK_TYPE" == "ROCE" ]; }; then
    echo 200000 > /proc/sys/vm/nr_hugepages
    export HCCL_IF_IP=$LOCAL_IP
    export GLOO_SOCKET_IFNAME=$NIC_NAME
    export TP_SOCKET_IFNAME=$NIC_NAME
    export HCCL_SOCKET_IFNAME=$NIC_NAME
    export HCCL_INTRA_ROCE_ENABLE=1

elif [ "$HARDWARE_SERIES" == "A3" ] && [ "$LINK_TYPE" == "HCCS" ]; then
    export ACL_OP_INIT_MODE=1
    export ASCEND_ENABLE_USE_FABRIC_MEM=1
elif [ "$HARDWARE_SERIES" == "A5" ]; then
    # A5 UBOE
    export ASCEND_GLOBAL_RESOURCE_CONFIG='{"comm_resource_config.protocol_desc":["uboe:device"]}'
    # A5 UB
    export ASCEND_LOCAL_COMM_RES='{"version":"1.3"}'
else
    echo "Error: Invalid HARDWARE_SERIES. Set to 'A2', 'A3', or 'A5'."
    exit 1
fi

source /usr/local/Ascend/ascend-toolkit/set_env.sh
source /usr/local/Ascend/nnal/atb/set_env.sh

KV_CONFIG='{
  "kv_connector": "MultiConnector",
  "kv_role": "'$KV_ROLE'",
  "kv_connector_extra_config": {
    "connectors": [
      {
        "kv_connector": "MooncakeConnectorV1",
        "kv_role": "'$KV_ROLE'",
        "kv_port": "'$KV_PORT'",
        "kv_connector_extra_config": {
          "prefill": {
            "dp_size": '$DATA_PARALLEL_SIZE',
            "tp_size": '$TENSOR_PARALLEL_SIZE'
          },
          "decode": {
            "dp_size": '$DATA_PARALLEL_SIZE',
            "tp_size": '$TENSOR_PARALLEL_SIZE'
          }
        }
      },
      {
        "kv_connector": "AscendStoreConnector",
        "kv_role": "'$KV_ROLE'",
        "kv_connector_extra_config": {
          "backend": "mooncake",
          "lookup_rpc_port": "'$LOOKUP_RPC_PORT'"
        }
      }
    ]
  }
}'

CMD_ARGS=(
  --model "$MODEL_PATH"
  --served-model-name "$SERVED_MODEL_NAME"
  --trust-remote-code
  --enforce-eager
  --data-parallel-size "$DATA_PARALLEL_SIZE"
  --tensor-parallel-size "$TENSOR_PARALLEL_SIZE"
  --port "$API_PORT"
  --max-num-seqs 20
  --max-model-len 32768
  --max-num-batched-tokens 16384
  --gpu-memory-utilization 0.9
  --kv-transfer-config "$KV_CONFIG"
)

python -m vllm.entrypoints.openai.api_server "${CMD_ARGS[@]}" > log_${ROLE}.log 2>&1

echo "vLLM started. Log file: log_${ROLE}.log"
```

启动 proxy_server（连接 Prefill 和 Decode 节点）：

```shell
python vllm-ascend/examples/disaggregated_prefill_v1/load_balance_proxy_server_example.py \
    --host localhost \
    --prefiller-hosts localhost \
    --prefiller-ports 8100 \
    --decoder-hosts localhost \
    --decoder-ports 8200
```

将 localhost 替换为实际 IP 地址。

执行推理：

短问题：

```shell
curl -s http://localhost:8000/v1/completions -H "Content-Type: application/json" -d '{ "model": "qwen3", "prompt": "Hello. I have a question. The president of the United States is", "max_completion_tokens": 200, "temperature":0.0 }'
```

长问题：

```shell
curl -s http://localhost:8000/v1/completions -H "Content-Type: application/json" -d '{ "model": "qwen3", "prompt": "Given the accelerating impacts of climate change\u2014including rising sea levels, increasing frequency of extreme weather events, loss of biodiversity, and adverse effects on agriculture and human health\u2014there is an urgent need for a robust, globally coordinated response. However, international efforts are complicated by a range of factors: economic disparities between high-income and low-income countries, differing levels of industrialization, varying access to clean energy technologies, and divergent political systems that influence climate policy implementation. In this context, how can global agreements like the Paris Accord be redesigned or strengthened to not only encourage but effectively enforce emission reduction targets? Furthermore, what mechanisms can be introduced to promote fair and transparent technology transfer, provide adequate financial support for climate adaptation in vulnerable regions, and hold nations accountable without exacerbating existing geopolitical tensions or disproportionately burdening those with historically lower emissions?", "max_completion_tokens": 256, "temperature":0.0 }'
```

MLA 模型如需支持 Decode 节点存储 KV Cache 供 Prefill 使用，则在 `AscendStoreConnector` 中添加 `consumer_is_to_put: true`；若 Prefill 节点启用 PP，还需设置 `prefill_pp_size` 或 `prefill_pp_layer_partition`：

```json
{
    "kv_connector": "AscendStoreConnector",
    "kv_role": "kv_consumer",
    "kv_load_failure_policy": "recompute",
    "kv_connector_extra_config": {
        "lookup_rpc_port": "0",
        "backend": "mooncake",
        "consumer_is_to_put": true,
        "prefill_pp_size": 2,
        "prefill_pp_layer_partition": "30,31"
    }
}
```

预期输出：返回符合 OpenAI Completions API 规范的 JSON 响应，包含 `id`、`choices`（含 `text` 与 `finish_reason`）、`usage` 等字段。

##### 步骤 4：PD-Mixed 场景

**pd_mix.sh：**

```shell
#!/bin/bash

# A2 (800I/800T A2) or A3 (800I/800T A3) or A5 (950PR/950DT)
HARDWARE_SERIES="A2"
# Link type: ROCE or HCCS in A3 series.
LINK_TYPE="ROCE"
LOCAL_IP="xx.xx.xx.xx"
NIC_NAME="xxxxxx"

MODEL_PATH="xxxxxxx/Qwen3-32B"
SERVED_MODEL_NAME="qwen3"
DATA_PARALLEL_SIZE=1
TENSOR_PARALLEL_SIZE=8
export ASCEND_RT_VISIBLE_DEVICES=0,1,2,3,4,5,6,7

# parameters required for kv pool and mooncake
export PYTHONHASHSEED=0
export MOONCAKE_CONFIG_PATH="/xxxxxx/mooncake.json"
export LD_LIBRARY_PATH=/usr/local/Ascend/ascend-toolkit/latest/python/site-packages/mooncake:$LD_LIBRARY_PATH

echo "Starting vLLM on Series: $HARDWARE_SERIES"

rm -rf /root/ascend/log/*
rm -rf ./connector.log

# 详细参数说明见「配置参数」章节
if [ "$HARDWARE_SERIES" == "A2" ] || { [ "$HARDWARE_SERIES" == "A3" ] && [ "$LINK_TYPE" == "ROCE" ]; }; then
    echo 200000 > /proc/sys/vm/nr_hugepages
    export HCCL_IF_IP=$LOCAL_IP
    export GLOO_SOCKET_IFNAME=$NIC_NAME
    export TP_SOCKET_IFNAME=$NIC_NAME
    export HCCL_SOCKET_IFNAME=$NIC_NAME
    export HCCL_INTRA_ROCE_ENABLE=1

elif [ "$HARDWARE_SERIES" == "A3" ] && [ "$LINK_TYPE" == "HCCS" ]; then
    export ACL_OP_INIT_MODE=1
    export ASCEND_ENABLE_USE_FABRIC_MEM=1
elif [ "$HARDWARE_SERIES" == "A5" ]; then
    # A5 UBOE
    export ASCEND_GLOBAL_RESOURCE_CONFIG='{"comm_resource_config.protocol_desc":["uboe:device"]}'
    # A5 UB
    export ASCEND_LOCAL_COMM_RES='{"version":"1.3"}'
else
    echo "Error: Invalid HARDWARE_SERIES. Set to 'A2', 'A3', or 'A5'."
    exit 1
fi

source /usr/local/Ascend/ascend-toolkit/set_env.sh
source /usr/local/Ascend/nnal/atb/set_env.sh

KV_CONFIG='{
  "kv_connector": "AscendStoreConnector",
  "kv_role": "kv_both",
  "kv_connector_extra_config": {
     "backend": "mooncake",
     "lookup_rpc_port": "0"
     }
}'

CMD_ARGS=(
  --model "$MODEL_PATH"
  --served-model-name "$SERVED_MODEL_NAME"
  --trust-remote-code
  --enforce-eager
  --data-parallel-size "$DATA_PARALLEL_SIZE"
  --tensor-parallel-size "$TENSOR_PARALLEL_SIZE"
  --port 8100
  --max-num-seqs 20
  --max-model-len 32768
  --max-num-batched-tokens 16384
  --gpu-memory-utilization 0.9
  --kv-transfer-config "$KV_CONFIG"
)

python -m vllm.entrypoints.openai.api_server "${CMD_ARGS[@]}" > log_mix.log 2>&1

echo "vLLM started. Log file: log_mix.log"
```

执行推理（无需单独启动 proxy，请求直接发送到混合部署端口）：

短问题：

```shell
curl -s http://localhost:8100/v1/completions -H "Content-Type: application/json" -d '{ "model": "qwen3", "prompt": "Hello. I have a question. The president of the United States is", "max_completion_tokens": 200, "temperature":0.0 }'
```

长问题：

```shell
curl -s http://localhost:8100/v1/completions -H "Content-Type: application/json" -d '{ "model": "qwen3", "prompt": "Given the accelerating impacts of climate change\u2014including rising sea levels, increasing frequency of extreme weather events, loss of biodiversity, and adverse effects on agriculture and human health\u2014there is an urgent need for a robust, globally coordinated response. However, international efforts are complicated by a range of factors: economic disparities between high-income and low-income countries, differing levels of industrialization, varying access to clean energy technologies, and divergent political systems that influence climate policy implementation. In this context, how can global agreements like the Paris Accord be redesigned or strengthened to not only encourage but effectively enforce emission reduction targets? Furthermore, what mechanisms can be introduced to promote fair and transparent technology transfer, provide adequate financial support for climate adaptation in vulnerable regions, and hold nations accountable without exacerbating existing geopolitical tensions or disproportionately burdening those with historically lower emissions?", "max_completion_tokens": 256, "temperature":0.0 }'
```

预期输出：返回 OpenAI Completions API 格式的 JSON 响应。

**注：** 对于启用了 `ASCEND_BUFFER_POOL` 的 MooncakeStore，建议在实际性能基准测试前执行预热阶段。由于 HCCS 单边通信连接在实例启动后延迟创建，全连接需要一次性时间开销（每连接 4 MB 设备内存）。预热建议：输入序列长度 8k、输出序列长度 1，请求总数 2-3x 设备数。

```shell
# 示例预热请求
curl -s http://localhost:8100/v1/completions -H "Content-Type: application/json" -d '{ "model": "qwen3", "prompt": "Hello.", "max_completion_tokens": 1, "temperature":0.0 }'
```

##### 步骤 5：MooncakeStore SSD Offload（Embedded Real Client 模式）

Embedded Real Client 模式（Mode A）下，`AscendStoreConnector`/`MooncakeBackend` 在 vLLM 启动时自动使用 `mooncake.json` 中的设置调用 `MooncakeDistributedStore.setup()`，无需单独 `mooncake_client` 进程。

SSD 磁盘使用控制环境变量：

```shell
# 800 GB 总磁盘、8 TP rank，约 100 GB per rank
export MOONCAKE_OFFLOAD_TOTAL_SIZE_LIMIT_BYTES=$((100 * 1024 * 1024 * 1024))
export MOONCAKE_OFFLOAD_BUCKET_MAX_TOTAL_SIZE=$((100 * 1024 * 1024 * 1024))
export MOONCAKE_OFFLOAD_BUCKET_EVICTION_POLICY=lru
export MOONCAKE_OFFLOAD_LOCAL_BUFFER_SIZE_BYTES=1073741824   # 1 GB
```

`MOONCAKE_OFFLOAD_LOCAL_BUFFER_SIZE_BYTES` 风险：若未对齐 1 GB（A3 + FabricMem 场景），可能导致 `adxl MallocMem` 失败或 `FileStorage init` 段错误。务必设置为 1 GB 整数倍。

#### 场景二：Memcache 后端

##### 步骤 1：前置检查

**检查内存：**

```shell
free -h
# 若缓存影响 KV cache pool 大小：
echo 3 > /proc/sys/vm/drop_caches
echo 1 > /proc/sys/vm/compact_memory
```

**A3 专属：扫描可用内存：**

```shell
python3 mem_scan.py                   # 1GB 规格扫描
python3 mem_scan.py -m 2              # 2MB 大页扫描
```

脚本地址：[mem_scan.py](https://gitcode.com/Ascend/memfabric_hybrid/blob/develop/script/mem_scan.py)

**Ascend 950 Products 专属（关闭签名验证 + 容器挂载 + 安装内核包）：**

```shell
# Step 1: 关闭 HDK 签名验证（每台裸机只需执行一次）
for i in {0..7}; do npu-smi set -t custom-op-secverify-enable -i $i -d 1; done;
for i in {0..7}; do npu-smi set -t custom-op-secverify-mode -i $i -d 0; done;
```

Docker 容器需挂载关键路径，示例命令：

```shell
docker run -u root -it -d --name ${NAME} --net=host --privileged=true \
    --device=/dev/davinci_manager --device=/dev/hisi_hdc --device=/dev/ummu --device=/dev/uburma \
    --device=/dev/davinci0 --device=/dev/davinci1 --device=/dev/davinci2 --device=/dev/davinci3 \
    --device=/dev/davinci4 --device=/dev/davinci5 --device=/dev/davinci6 --device=/dev/davinci7 \
    -v /usr/bin/urma_admin:/usr/bin/urma_admin \
    -v /lib/route.conf:/lib/route.conf \
    -v /etc/hccl_rootinfo.json:/etc/hccl_rootinfo.json \
    -v /usr/local/sbin/npu-smi:/usr/local/sbin/npu-smi \
    -v /usr/local/sbin:/usr/local/sbin \
    -v /usr/local/dcmi:/usr/local/dcmi \
    -v /var/log/npu/:/usr/slog \
    -v /etc/hccn.conf:/etc/hccn.conf \
    -v /etc/hixlep:/etc/hixlep \
    -v /usr/local/Ascend/driver:/usr/local/Ascend/driver \
    -w /home \
    ${IMAGES_ID} \
    bash
```

容器内需更新 `/lib/route.conf`。

**启用 SSD 前检查磁盘状态：**

```shell
lsblk /dev/nvme1n1                    # 无分区
mount | grep nvme1n1                  # 无挂载点
blkid /dev/nvme1n1                    # 无文件系统签名
```

若无物理磁盘，可使用 Loop Device 模拟：

```shell
dd if=/dev/zero of=/data/boostio_disk.img bs=1G count=640 status=progress
LOOP_DEV=$(losetup --find --show --direct-io=on /data/boostio_disk.img)
echo "${LOOP_DEV}"
```

##### 步骤 2：软件安装

```shell
pip install memfabric-hybrid
pip install memcache-hybrid
```

启用 Memcache SSD Cache 需 `memcache_hybrid >= 1.2.0`。

##### 步骤 3：配置 Memcache 配置文件

查找安装路径：

```shell
pip show memcache_hybrid
```

以 `{INSTALL_PATH}` 表示输出中的 `Location` 值。

**mmc-meta.conf：**

```ini
ock.mmc.meta_service_url = tcp://xx.xx.xx.xx:5000
ock.mmc.meta_service.config_store_url = tcp://xx.xx.xx.xx:6000
ock.mmc.meta_service.metrics_url = http://xx.xx.xx.xx:8000
ock.mmc.log_level = info
# 启用 SSD 时调优以下参数以提高 SSD cache 命中率
ock.mmc.evict_threshold_high = 70
ock.mmc.evict_threshold_low = 60
ock.mmc.rewarm.dram_watermark = 95
```

**mmc-local.conf：**

```ini
ock.mmc.meta_service_url = tcp://xx.xx.xx.xx:5000
ock.mmc.local_service.config_store_url = tcp://xx.xx.xx.xx:6000
ock.mmc.log_level = info
ock.mmc.local_service.world_size = 256
ock.mmc.local_service.protocol = device_sdma
ock.mmc.local_service.dram.size = 1GB
ock.mmc.local_service.max.dram.size = 1024GB
# SSD feature related parameters below
ock.mmc.local_service.storage.enabled = false
ubsio.disk.path = /dev/nvmexn1:/dev/nvmexn2p1:/dev/loopX
ubsio.mem.size_in_gb = 10
ubsio.standalone.device_count = 8
ubsio.standalone.force_new_disk = true
```

##### 步骤 4：运行 MetaService

```shell
export MMC_META_CONFIG_PATH={INSTALL_PATH}/memcache_hybrid/config/mmc-meta.conf

python -c "from memcache_hybrid import MetaService; MetaService.main()"
```

预期输出：MetaService 正常启动无报错。

##### 步骤 5：PD 分离场景

**run_prefill.sh / run_decode.sh：**

```shell
#!/bin/bash

# prefill / decode
ROLE="prefill"
# A2 (800I/800T A2) or A3 (800I/800T A3) or A5 (950PR/950DT)
HARDWARE_SERIES="A2"
# Link type: ROCE or HCCS in A3 series.
LINK_TYPE="ROCE"
LOCAL_IP="xx.xx.xx.xx"
NIC_NAME="xxxxxx"

MODEL_PATH="xxxxxxx/Qwen3-32B"
SERVED_MODEL_NAME="qwen3"
DATA_PARALLEL_SIZE=1
TENSOR_PARALLEL_SIZE=8
export ASCEND_RT_VISIBLE_DEVICES=0,1,2,3,4,5,6,7

# parameters required for kv pool and memcache
export PYTHONHASHSEED=0
export MMC_LOCAL_CONFIG_PATH={INSTALL_PATH}/memcache_hybrid/config/mmc-local.conf
export LD_LIBRARY_PATH={INSTALL_PATH}/memcache_hybrid/lib:${PYTHON_LIB_DIR}:${LD_LIBRARY_PATH}

if [ "$ROLE" == "prefill" ]; then
    KV_ROLE="kv_producer"
    KV_PORT="20001"
    LOOKUP_RPC_PORT="0"
    API_PORT="8100"
else
    KV_ROLE="kv_consumer"
    KV_PORT="20002"
    LOOKUP_RPC_PORT="1"
    API_PORT="8200"
fi

echo "Starting vLLM on Series: $HARDWARE_SERIES, Role: $ROLE"

rm -rf /root/ascend/log/*
rm -rf ./connector.log

# 详细参数说明见「配置参数」章节
if [ "$HARDWARE_SERIES" == "A2" ] || { [ "$HARDWARE_SERIES" == "A3" ] && [ "$LINK_TYPE" == "ROCE" ]; }; then
    echo 200000 > /proc/sys/vm/nr_hugepages
    export HCCL_IF_IP=$LOCAL_IP
    export GLOO_SOCKET_IFNAME=$NIC_NAME
    export TP_SOCKET_IFNAME=$NIC_NAME
    export HCCL_SOCKET_IFNAME=$NIC_NAME
    export HCCL_INTRA_ROCE_ENABLE=1

elif [ "$HARDWARE_SERIES" == "A3" ] && [ "$LINK_TYPE" == "HCCS" ]; then
    export ACL_OP_INIT_MODE=1
    export ASCEND_ENABLE_USE_FABRIC_MEM=1
elif [ "$HARDWARE_SERIES" == "A5" ]; then
    # A5 UBOE
    export ASCEND_GLOBAL_RESOURCE_CONFIG='{"comm_resource_config.protocol_desc":["uboe:device"]}'
    # A5 UB
    export ASCEND_LOCAL_COMM_RES='{"version":"1.3"}'
else
    echo "Error: Invalid HARDWARE_SERIES. Set to 'A2', 'A3', or 'A5'."
    exit 1
fi

source /usr/local/Ascend/ascend-toolkit/set_env.sh
source /usr/local/Ascend/nnal/atb/set_env.sh

KV_CONFIG='{
  "kv_connector": "MultiConnector",
  "kv_role": "'$KV_ROLE'",
  "kv_connector_extra_config": {
    "connectors": [
      {
        "kv_connector": "MooncakeConnectorV1",
        "kv_role": "'$KV_ROLE'",
        "kv_port": "'$KV_PORT'",
        "kv_connector_extra_config": {
          "prefill": {
            "dp_size": '$DATA_PARALLEL_SIZE',
            "tp_size": '$TENSOR_PARALLEL_SIZE'
          },
          "decode": {
            "dp_size": '$DATA_PARALLEL_SIZE',
            "tp_size": '$TENSOR_PARALLEL_SIZE'
          }
        }
      },
      {
        "kv_connector": "AscendStoreConnector",
        "kv_role": "'$KV_ROLE'",
        "kv_connector_extra_config": {
          "backend": "memcache",
          "lookup_rpc_port": "'$LOOKUP_RPC_PORT'",
          "use_layerwise": false
        }
      }
    ]
  }
}'

CMD_ARGS=(
  --model "$MODEL_PATH"
  --served-model-name "$SERVED_MODEL_NAME"
  --trust-remote-code
  --enforce-eager
  --data-parallel-size "$DATA_PARALLEL_SIZE"
  --tensor-parallel-size "$TENSOR_PARALLEL_SIZE"
  --port "$API_PORT"
  --max-num-seqs 20
  --max-model-len 32768
  --max-num-batched-tokens 16384
  --gpu-memory-utilization 0.9
  --kv-transfer-config "$KV_CONFIG"
)

python -m vllm.entrypoints.openai.api_server "${CMD_ARGS[@]}" > log_${ROLE}.log 2>&1

echo "vLLM started. Log file: log_${ROLE}.log"
```

`use_layerwise` 仅 Prefill 节点可设为 `true` 启用逐层 KV 存取，需要 Memcache 后端支持。`consumer_is_to_put` 和 `consumer_is_to_load` 同样可通过 `kv_connector_extra_config` 配置。

启动 proxy_server 与执行推理请参考「场景一：Mooncake 后端 步骤 3：PD 分离场景」中的对应子步骤。

预期输出：同 Mooncake 场景，返回 OpenAI Completions API JSON 响应。

##### 步骤 6：PD-Mixed 场景

**pd_mix.sh：**

```shell
#!/bin/bash

# A2 (800I/800T A2) or A3 (800I/800T A3) or A5 (950PR/950DT)
HARDWARE_SERIES="A2"
# Link type: ROCE or HCCS in A3 series.
LINK_TYPE="ROCE"
LOCAL_IP="xx.xx.xx.xx"
NIC_NAME="xxxxxx"

MODEL_PATH="xxxxxxx/Qwen3-32B"
SERVED_MODEL_NAME="qwen3"
DATA_PARALLEL_SIZE=1
TENSOR_PARALLEL_SIZE=8
export ASCEND_RT_VISIBLE_DEVICES=0,1,2,3,4,5,6,7

# parameters required for kv pool and memcache
export PYTHONHASHSEED=0
export MMC_LOCAL_CONFIG_PATH={INSTALL_PATH}/memcache_hybrid/config/mmc-local.conf
export LD_LIBRARY_PATH={INSTALL_PATH}/memcache_hybrid/lib:${PYTHON_LIB_DIR}:${LD_LIBRARY_PATH}

echo "Starting vLLM on Series: $HARDWARE_SERIES"

rm -rf /root/ascend/log/*
rm -rf ./connector.log

# 详细参数说明见「配置参数」章节
if [ "$HARDWARE_SERIES" == "A2" ] || { [ "$HARDWARE_SERIES" == "A3" ] && [ "$LINK_TYPE" == "ROCE" ]; }; then
    echo 200000 > /proc/sys/vm/nr_hugepages
    export HCCL_IF_IP=$LOCAL_IP
    export GLOO_SOCKET_IFNAME=$NIC_NAME
    export TP_SOCKET_IFNAME=$NIC_NAME
    export HCCL_SOCKET_IFNAME=$NIC_NAME
    export HCCL_INTRA_ROCE_ENABLE=1

elif [ "$HARDWARE_SERIES" == "A3" ] && [ "$LINK_TYPE" == "HCCS" ]; then
    export ACL_OP_INIT_MODE=1
    export ASCEND_ENABLE_USE_FABRIC_MEM=1
elif [ "$HARDWARE_SERIES" == "A5" ]; then
    # A5 UBOE
    export ASCEND_GLOBAL_RESOURCE_CONFIG='{"comm_resource_config.protocol_desc":["uboe:device"]}'
    # A5 UB
    export ASCEND_LOCAL_COMM_RES='{"version":"1.3"}'
else
    echo "Error: Invalid HARDWARE_SERIES. Set to 'A2', 'A3', or 'A5'."
    exit 1
fi

source /usr/local/Ascend/ascend-toolkit/set_env.sh
source /usr/local/Ascend/nnal/atb/set_env.sh

KV_CONFIG='{
  "kv_connector": "AscendStoreConnector",
  "kv_role": "kv_both",
  "kv_connector_extra_config": {
     "backend": "memcache",
     "lookup_rpc_port": "0",
     "use_layerwise": false
  }
}'

CMD_ARGS=(
  --model "$MODEL_PATH"
  --served-model-name "$SERVED_MODEL_NAME"
  --trust-remote-code
  --enforce-eager
  --data-parallel-size "$DATA_PARALLEL_SIZE"
  --tensor-parallel-size "$TENSOR_PARALLEL_SIZE"
  --port 8100
  --max-num-seqs 20
  --max-model-len 32768
  --max-num-batched-tokens 16384
  --gpu-memory-utilization 0.9
  --kv-transfer-config "$KV_CONFIG"
)

python -m vllm.entrypoints.openai.api_server "${CMD_ARGS[@]}" > log_mix.log 2>&1

echo "vLLM started. Log file: log_mix.log"
```

执行推理参考「场景一：Mooncake 后端 步骤 4：PD-Mixed 场景」中的推理命令。

##### 步骤 7：Memcache 与 vLLM 分离部署

分离部署模式将 Memcache 运行在独立进程中，仅支持 A3 HCCS 场景。

步骤如下：

1. 启动 MetaService（同上）。
2. 使用以下配置的 `mmc-local-standalone.conf` 在每节点启动独立 Memcache 进程：

   ```ini
   ock.mmc.local_service.dram.size = 600GB
   ock.mmc.local_service.max.dram.size = 1024GB
   ```

3. 等待所有节点上报初始化成功。
4. 使用以下配置的 `mmc-local.conf`（`dram.size = 0GB`）启动 vLLM：

   ```ini
   ock.mmc.local_service.dram.size = 0GB
   ock.mmc.local_service.max.dram.size = 1024GB
   ```

启动脚本参考：[Memcache + vLLM + A3 分离部署案例](https://gitcode.com/Ascend/memcache/wiki/MemCache+vLLM+A3%E5%88%86%E7%A6%BB%E9%83%A8%E7%BD%B2%E6%A1%88%E4%BE%8B.md)

##### 步骤 8：启用 Memcache SSD Cache

启用 `mmc-local-standalone.conf` 中的 SSD 相关参数，并参考 UBS IO 内存池计算公式调整 `ubsio.mem.size_in_gb`：

```text
maximum ubsio.mem.size_in_gb = min(3072, floor(available node memory for UBS IO (GB) / number of DRAM-enabled local services))
```

#### 场景三：Yuanrong 后端

##### 步骤 1：安装 Yuanrong Datasystem

```bash
pip install openyuanrong-datasystem
python -c "import yr.datasystem; print('Yuanrong Datasystem is ready')"
dscli --version
```

预期输出：
- `Yuanrong Datasystem is ready`
- 正常显示 `dscli` 版本号。

若预编译包与 CANN 或驱动版本不匹配，请从源码构建 Yuanrong Datasystem：[Yuanrong Datasystem](https://atomgit.com/openeuler/yuanrong-datasystem)。

##### 步骤 2：选择服务发现后端

**Option 1：启动 Coordinator**

```bash
COORDINATOR_ADDRESS="<coordinator_ip>:31511"

dscli start -c \
  --coordinator_address "${COORDINATOR_ADDRESS}"
```

预期启动打印 `Start coordinator service ... success`。

单节点快速启动（Coordinator + Worker 一步完成）：

```bash
dscli start -a \
  --coordinator_address "127.0.0.1:31511" \
  --worker_address "127.0.0.1:31501" \
  --shared_memory_size_mb 4096
```

**Option 2：启动 etcd**

```bash
ETCD_VERSION="v3.5.12"
ETCD_IP="127.0.0.1"
if [ "$(uname -m)" = "aarch64" ]; then
  ETCD_ARCH="linux-arm64"
else
  ETCD_ARCH="linux-amd64"
fi
wget https://github.com/etcd-io/etcd/releases/download/${ETCD_VERSION}/etcd-${ETCD_VERSION}-${ETCD_ARCH}.tar.gz
tar -xvf etcd-${ETCD_VERSION}-${ETCD_ARCH}.tar.gz
cd etcd-${ETCD_VERSION}-${ETCD_ARCH}
sudo cp etcd etcdctl /usr/local/bin/

etcd \
  --name etcd-single \
  --data-dir /tmp/etcd-data \
  --listen-client-urls http://0.0.0.0:2379 \
  --advertise-client-urls http://${ETCD_IP}:2379 \
  --listen-peer-urls http://0.0.0.0:2380 \
  --initial-advertise-peer-urls http://${ETCD_IP}:2380 \
  --initial-cluster etcd-single=http://${ETCD_IP}:2380 &

etcdctl --endpoints "${ETCD_IP}:2379" put key "value"
etcdctl --endpoints "${ETCD_IP}:2379" get key
```

预期输出：`etcdctl put key "value"` 返回 `OK`；`etcdctl get key` 返回 `key`->`value`。

##### 步骤 3：启动 Datasystem Worker

```bash
COORDINATOR_ADDRESS="<coordinator_ip>:31511"
WORKER_IP="<worker_ip>"
WORKER_LOG_DIR="/var/log/yuanrong/worker"
sudo mkdir -p "${WORKER_LOG_DIR}"
sudo chown "$(id -u):$(id -g)" "${WORKER_LOG_DIR}"

dscli start -w \
  --worker_address "${WORKER_IP}:31501" \
  --coordinator_address "${COORDINATOR_ADDRESS}" \
  --log_dir "${WORKER_LOG_DIR}" \
  --shared_memory_size_mb 40960 \
  --arena_per_tenant 1 \
  --enable_huge_tlb true \
  --enable_fallocate false \
  --rpc_thread_num 64 \
  --oc_thread_num 64 \
  --enable_worker_worker_batch_get true \
  --sc_regular_socket_num 0 \
  --sc_stream_socket_num 0
```

预期输出：Worker 正常启动无报错，日志写入 `--log_dir` 目录。

多节点部署时，每个节点运行一个 Worker，使用唯一可到达的 `worker_address`，所有 Worker 使用相同的服务发现后端和地址。

##### 步骤 4：配置环境变量与 `yuanrong.json`

```bash
export PYTHONHASHSEED=0
export DS_WORKER_ADDR="${WORKER_IP}:31501"
export DATASYSTEM_CLIENT_LOG_DIR="/var/log/yuanrong/client"
export DS_ENABLE_EXCLUSIVE_CONNECTION=0
export DS_ENABLE_REMOTE_H2D=0
```

配置 `yuanrong.json`（由 `YR_CONFIG_PATH` 指向）：

```json
{
    "worker_addr": "xx.xx.xx.xx:31501",
    "connect_timeout_ms": 9000,
    "request_timeout_ms": 0,
    "get_sub_timeout_ms": 0,
    "enable_remote_h2d": false,
    "remote_h2d_transport_backend": "HIXL",
    "enable_fabric_mem": false,
    "enable_dev_mem_pregister": false,
    "use_layerwise": false
}
```

`worker_addr` 必须与本地 `dscli start --worker_address` 值一致。

##### 步骤 5：运行 AscendStoreConnector（Yuanrong 后端）

```bash
python3 -m vllm.entrypoints.openai.api_server \
    --model /xxxxx/Qwen2.5-7B-Instruct \
    --port 8100 \
    --trust-remote-code \
    --enforce-eager \
    --no-enable-prefix-caching \
    --tensor-parallel-size 1 \
    --data-parallel-size 1 \
    --max-model-len 10000 \
    --block-size 128 \
    --max-num-batched-tokens 4096 \
    --kv-transfer-config \
    '{
    "kv_connector": "AscendStoreConnector",
    "kv_role": "kv_both",
    "kv_load_failure_policy": "recompute",
    "kv_connector_extra_config": {
        "lookup_rpc_port": "1",
        "backend": "yuanrong",
        "use_layerwise": false
    }
}'
```

`lookup_rpc_port` 为 Pooling Scheduler 进程与 Worker 进程之间的 RPC 端口，每个实例必须使用唯一值。

**注：** Yuanrong 后端在调用 Datasystem 前会规范化 KV 键。支持的 ASCII 键（最长 1024 字节）保持不变；更长的键或包含不支持的字符的键会被重写为最多 1024 字符并附加 hash 后缀，因此调试后端存储时不应依赖原始键字符串。无需额外的 buffer 预注册步骤。

预期输出：vLLM 正常启动，OpenAI API 服务就绪。

### 验证特性（必选）

**内容缺失，需要补充。以下内容基于原文上下文推断，请人工确认：**

| 验证项 | 验证命令 | 预期输出 |
| :--- | :--- | :--- |
| vLLM 推理服务 | `curl -s http://localhost:<port>/v1/completions ...` | 返回包含 `choices` 字段的 JSON 响应，`finish_reason` 为 `stop` 或 `length` |
| Yuanrong Datasystem 就绪 | `python -c "import yr.datasystem; print('Yuanrong Datasystem is ready')"` | 打印 `Yuanrong Datasystem is ready` |
| dscli 可用性 | `dscli --version` | 正常显示版本号 |
| Coordinator 启动 | `dscli start -c ...` | 打印 `Start coordinator service ... success` |
| etcd 可用性 | `etcdctl put key "value"; etcdctl get key` | put 返回 `OK`，get 返回 `key`->`value` |
| Worker 日志 | 检查 `--log_dir` 下的 Worker 日志 | 无错误/异常退出 |
| vLLM 启动日志 | `cat log_<role>.log` | 可见 `Available KV cache memory` 等信息，无堆栈报错 |
| SSD Offload 缓冲区 | 启动时日志 | 每个 rank 打印 `AlignedClientBufferAllocator: allocated <N> bytes` |
| Mooncake master | `curl -s http://<master_host>:9003/api/v1/tenant_quotas` | 返回租户配额 JSON |

## 配置参数（必选）

**依据原文上下文内容重组，请进行人工校验。**

### kv-transfer-config 通用参数

| 参数 | 类型 | 默认值 | 必填 | 取值范围 | 说明 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| kv_load_failure_policy | 内容缺失，需要人工补齐。 | fail | 否 | 内容缺失，需要人工补齐。 | KV 加载失败时的处理策略：`recompute` 回退重算（暂不支持混合注意力模型如 DeepSeekV4、Qwen 3.5），`fail` 直接终止请求。使用 MultiConnector 时需在顶层 `kv-transfer-config` 配置。 |
| lookup_rpc_port | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | Pooling Scheduler 进程与 Worker 进程之间的 RPC 通信端口，每个实例需配置唯一端口。 |
| load_async | 内容缺失，需要人工补齐。 | false | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是否启用异步加载。 |
| backend | 内容缺失，需要人工补齐。 | mooncake | 内容缺失，需要人工补齐。 | mooncake / memcache / yuanrong | KV Pool 存储后端。 |
| consumer_is_to_put | 内容缺失，需要人工补齐。 | false | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | Decode 节点是否将 KV Cache 存入 KV Pool。 |
| consumer_is_to_load | 内容缺失，需要人工补齐。 | false | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | Decode 节点是否从 KV Pool 加载 KV Cache。 |
| use_layerwise | 内容缺失，需要人工补齐。 | false | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 逐层 KV 存取，仅 Prefill 节点支持，需 Memcache 后端。 |
| prefill_pp_size | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | Prefill 节点启用 PP 时必填 | 内容缺失，需要人工补齐。 | Prefill PP 大小。 |
| prefill_pp_layer_partition | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | Prefill PP 层划分。 |
| qos_priority | 内容缺失，需要人工补齐。 | 0 | 否 | [0, 4]（整数，越大优先级越高） | KV Pool 传输 QoS 优先级。 |

### mooncake.json 参数

| 参数 | 类型 | 默认值 | 必填 | 取值范围 | 说明 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| metadata_server | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是 | P2PHANDSHAKE | 配置为 P2PHANDSHAKE。 |
| protocol | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是 | ascend | NPU 上必须设为 ascend。 |
| device_name | 内容缺失，需要人工补齐。 | ""（空字符串） | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | Ascend 协议不使用设备名称，留空。 |
| master_server_address | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是 | `<ip>:<port>` | Master 服务 IP 和端口。可通过 `MOONCAKE_MASTER` 环境变量覆盖。 |
| global_segment_size | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是 | 需对齐 1 GB（1024 MB / 1048576 KB / 1073741824 B） | 每张卡注册到 KV Pool 的内存大小。可通过 `MOONCAKE_GLOBAL_SEGMENT_SIZE` 环境变量覆盖。 |
| preferred_segment | 内容缺失，需要人工补齐。 | false | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是否优先将 KV 存在本地 segment。 |
| prefer_alloc_in_same_node | 内容缺失，需要人工补齐。 | true | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是否优先在本节点分配 KV。 |
| enable_ssd_offload | 内容缺失，需要人工补齐。 | false | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是否启用 SSD Offload。不支持环境变量配置。 |
| ssd_offload_path | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 当 enable_ssd_offload=true 时必填 | 绝对路径 | SSD Offload 数据存储目录的绝对路径。目录必须存在且可写；相对路径、符号链接、含 `..` 的路径均被拒绝。 |
| tenant_id | 内容缺失，需要人工补齐。 | default | 否 | 内容缺失，需要人工补齐。 | Mooncake 租户命名空间。非 default 租户需 Mooncake >= 0.3.12。 |

### mooncake_master 参数

| 参数 | 类型 | 默认值 | 必填 | 取值范围 | 说明 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| port | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是 | 内容缺失，需要人工补齐。 | Master 服务监听端口，需与 mooncake.json 中 `master_server_address` 的端口一致。 |
| eviction_high_watermark_ratio | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | 内容缺失，需要人工补齐。 | Mooncake Store 触发 eviction 的高水位阈值。 |
| eviction_ratio | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | 内容缺失，需要人工补齐。 | Eviction 时移除的存储对象比例。 |
| default_kv_lease_ttl | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | 内容缺失，需要人工补齐。 | KV 对象默认租约 TTL（毫秒），需大于 `ASCEND_CONNECT_TIMEOUT` 与 `ASCEND_TRANSFER_TIMEOUT`。 |
| enable_offload | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 仅 SSD offload 启用时需要 | 内容缺失，需要人工补齐。 | 设为 true 以启用 Master 端 SSD Offload。 |
| client_ttl | 内容缺失，需要人工补齐。 | 10 | 否 | 内容缺失，需要人工补齐。 | 客户端最后一次 Ping 后的保活秒数。 |
| enable_multi_tenants | 内容缺失，需要人工补齐。 | false | 否 | 内容缺失，需要人工补齐。 | 启用严格多租户模式。 |
| tenant_quota_connector_type | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 启用多租户时必填 | file / etcd | 租户配额连接器类型。 |
| tenant_quota_connector_uri | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 启用多租户时必填 | 内容缺失，需要人工补齐。 | 租户配额文件路径或 etcd endpoints。 |

### Mooncake SSD 环境变量

| 参数 | 类型 | 默认值 | 必填 | 取值范围 | 说明 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| MOONCAKE_OFFLOAD_LOCAL_BUFFER_SIZE_BYTES | 内容缺失，需要人工补齐。 | 1342177280（1280 MB） | 否 | A3 + `ASCEND_ENABLE_USE_FABRIC_MEM=1` 时需对齐 1 GB | 每 rank SSD 读写缓冲区大小（字节）。不可在 mooncake.json 中配置。`BUFFER_OVERFLOW` 时需增大。 |
| MOONCAKE_OFFLOAD_BUCKET_MAX_TOTAL_SIZE | 内容缺失，需要人工补齐。 | 0 | 否 | 内容缺失，需要人工补齐。 | Eviction 阈值（字节）。0 表示使用物理磁盘容量的 90%。 |
| MOONCAKE_OFFLOAD_BUCKET_EVICTION_POLICY | 内容缺失，需要人工补齐。 | none | 否 | none / fifo / lru | SSD eviction 策略。 |
| MOONCAKE_OFFLOAD_TOTAL_SIZE_LIMIT_BYTES | 内容缺失，需要人工补齐。 | 2199023255552（2 TB） | 强烈建议显式设置 | 需匹配真实磁盘容量 | 每 rank 向 Mooncake master 报告的最大磁盘使用量。默认值远超真实磁盘容量，需覆盖。 |

### Memcache 配置参数

**mmc-meta.conf：**

| 参数 | 类型 | 默认值 | 必填 | 取值范围 | 说明 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| ock.mmc.meta_service_url | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是 | tcp://<ip>:<port> | MetaService 地址，P 节点与 D 节点需配置相同的 endpoint。 |
| ock.mmc.meta_service.config_store_url | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是 | tcp://<ip>:<port> | 配置存储地址。 |
| ock.mmc.meta_service.metrics_url | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | http://<ip>:<port> | Metrics 地址。 |
| ock.mmc.log_level | 内容缺失，需要人工补齐。 | info | 否 | 内容缺失，需要人工补齐。 | 日志级别。 |
| ock.mmc.evict_threshold_high | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | 内容缺失，需要人工补齐。 | 启用 SSD 时提高 SSD cache 命中率的高水位阈值。 |
| ock.mmc.evict_threshold_low | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | 内容缺失，需要人工补齐。 | 启用 SSD 时提高 SSD cache 命中率的低水位阈值。 |
| ock.mmc.rewarm.dram_watermark | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | 内容缺失，需要人工补齐。 | 启用 SSD 时 DRAM 水位线。 |

**mmc-local.conf：**

| 参数 | 类型 | 默认值 | 必填 | 取值范围 | 说明 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| ock.mmc.meta_service_url | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是 | tcp://<ip>:<port> | 需与 mmc-meta.conf 中一致。 |
| ock.mmc.local_service.config_store_url | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是 | tcp://<ip>:<port> | 必须与 mmc-meta.conf 中 `ock.mmc.meta_service.config_store_url` 一致。 |
| ock.mmc.local_service.world_size | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是 | 内容缺失，需要人工补齐。 | 支持的 LocalService 最大数量（含将来加入的）。 |
| ock.mmc.local_service.protocol | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是 | device_rdma / device_sdma / device_urma / device_uboe | 通信协议。A2 推荐 `device_rdma`（RoCE）；A3 HCCS 推荐 `device_sdma`；A5 UB 设为 `device_urma`；A5 UBOE 设为 `device_uboe`。 |
| ock.mmc.local_service.dram.size | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是 | 0 或正数加单位（如 1GB、40GB） | 每 die 分配的 DRAM 大小。A3 HCCS 场景设为 0GB。 |
| ock.mmc.local_service.max.dram.size | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | 内容缺失，需要人工补齐。 | DRAM 最大值，当各 rank 贡献不同大小 DRAM 时需要。 |
| ock.mmc.local_service.storage.enabled | 内容缺失，需要人工补齐。 | false | 否 | 内容缺失，需要人工补齐。 | 启用 SSD 缓存。 |
| ubsio.disk.path | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | SSD 启用时必填 | 绝对路径，多路径以 `:` 分隔 | SSD 块设备路径。设备必须专用、无挂载点、无文件系统签名。不推荐 `/dev/sd*`。 |
| ubsio.mem.size_in_gb | 内容缺失，需要人工补齐。 | 10 | 否 | 整数 [0, 3072]；SSD 缓存至少 5 | 每进程 UBS IO 内存池大小（GB）。分离部署建议 50 GB。 |
| ubsio.standalone.device_count | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是 | 内容缺失，需要人工补齐。 | `dram.size` 不为 0 的 LocalService 数量。 |
| ubsio.standalone.force_new_disk | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | 内容缺失，需要人工补齐。 | 是否将 SSD 初始化为新盘（当前版本不支持故障恢复，建议设为 true）。 |

### Yuanrong dscli Worker 参数

| 参数 | 类型 | 默认值 | 必填 | 取值范围 | 说明 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| worker_address | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是 | `<host>:<port>` | Worker 地址，必须与 `yuanrong.json` 中 `worker_addr` 一致。 |
| coordinator_address | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | Option 1 | `<ip>:31511` | Coordinator 地址。 |
| etcd_address | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | Option 2 | `<ip>:2379` | etcd 地址。 |
| log_dir | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | 内容缺失，需要人工补齐。 | Worker 日志目录，使用绝对路径。 |
| shared_memory_size_mb | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是 | 内容缺失，需要人工补齐。 | 共享内存大小（MB）。示例 40960（40 GB）。 |
| arena_per_tenant | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | 内容缺失，需要人工补齐。 | 每租户共享内存 arena 数。保守建议为 1。 |
| enable_huge_tlb | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | 内容缺失，需要人工补齐。 | 是否使用 HugeTLB 后盾共享内存。 |
| enable_fallocate | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | 内容缺失，需要人工补齐。 | 是否对共享内存文件执行 fallocate。与 HugeTLB 配合使用时建议设为 false。 |
| rpc_thread_num | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | 内容缺失，需要人工补齐。 | RPC/ZMQ 服务并发数。 |
| oc_thread_num | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | 内容缺失，需要人工补齐。 | Object Cache 业务线程池大小。 |
| enable_worker_worker_batch_get | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | 内容缺失，需要人工补齐。 | 启用 Worker 间批量 Object Cache 读取。 |
| sc_regular_socket_num | 内容缺失，需要人工补齐。 | 0 | 否 | >= 0 | Stream Cache 常规 socket 数。KV Pool 不使用 Stream Cache 时保持为 0。 |
| sc_stream_socket_num | 内容缺失，需要人工补齐。 | 0 | 否 | >= 0 | Stream Cache stream socket 数。KV Pool 不使用 Stream Cache 时保持为 0。 |
| remote_h2d_device_ids | 内容缺失，需要人工补齐。 | 空 | 否 | 逗号分隔设备 ID，如 `"0,1,2,3,4,5,6,7"` | 非空则启用 Worker 端 Remote H2D。 |
| remote_h2d_link_type | 内容缺失，需要人工补齐。 | ROCE | 否 | ROCE / HCCS（区分大小写） | 链路类型。`ROCE` 对应客户端 `P2P_TRANSFER`；`HCCS` 对应客户端 `HIXL`。 |
| remote_h2d_hccs_buffer_pool | 内容缺失，需要人工补齐。 | 4:8 | 否 | `<count>:<size>` | HIXL buffer-pool 参数，仅 `link_type=HCCS` 时使用。 |

### Yuanrong 环境变量

| 参数 | 类型 | 默认值 | 必填 | 取值范围 | 说明 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| PYTHONHASHSEED | 内容缺失，需要人工补齐。 | 0 | 是 | 内容缺失，需要人工补齐。 | 所有节点必须一致以保证统一 hash 生成。 |
| DS_WORKER_ADDR | 内容缺失，需要人工补齐。 | N/A | 是 | `<host>:<port>` | Datasystem Worker 地址，须与本地 `dscli start --worker_address` 值一致。 |
| DATASYSTEM_CLIENT_LOG_DIR | 内容缺失，需要人工补齐。 | ~/.datasystem/logs | 否 | 内容缺失，需要人工补齐。 | Yuanrong 客户端 SDK 日志目录。 |
| DS_ENABLE_EXCLUSIVE_CONNECTION | 内容缺失，需要人工补齐。 | 0 | 否 | 内容缺失，需要人工补齐。 | 设为 1 启用排他连接模式。 |
| DS_ENABLE_REMOTE_H2D | 内容缺失，需要人工补齐。 | 0 | 否 | 内容缺失，需要人工补齐。 | 设为 1 启用 Remote H2D（需满足 Remote H2D 前提条件）。 |

### yuanrong.json 字段

| 参数 | 类型 | 默认值 | 必填 | 取值范围 | 说明 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| worker_addr | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 是 | `<host>:<port>` | Datasystem Worker 地址，需与 `dscli start --worker_address` 一致。 |
| connect_timeout_ms | 内容缺失，需要人工补齐。 | 9000 | 内容缺失，需要人工补齐。 | 整数 >= 500 | 连接建立超时（毫秒）。 |
| request_timeout_ms | 内容缺失，需要人工补齐。 | 0 | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 请求超时（毫秒）。0 表示使用 `connect_timeout_ms`。 |
| get_sub_timeout_ms | 内容缺失，需要人工补齐。 | 0 | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | `mget_h2d_from_multi_buffers` 等待对象就绪的超时（毫秒）。 |
| enable_remote_h2d | 内容缺失，需要人工补齐。 | false | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 传递给 `HeteroClient.enable_remote_h2d`。 |
| remote_h2d_transport_backend | 内容缺失，需要人工补齐。 | HIXL | 内容缺失，需要人工补齐。 | HIXL / P2P_TRANSFER | 客户端传输名称，对应 Worker 端 `--remote_h2d_link_type`（HCCS ↔ HIXL，ROCE ↔ P2P_TRANSFER）。 |
| enable_fabric_mem | 内容缺失，需要人工补齐。 | false | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 选择 HIXL FabricMem 模式。仅 `remote_h2d_transport_backend=HIXL` 时有意义。 |
| enable_dev_mem_pregister | 内容缺失，需要人工补齐。 | false | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 客户端设备内存预注册总开关。实际生效需 `enable_remote_h2d=true`、`remote_h2d_transport_backend=HIXL` 且 `enable_fabric_mem=false`。 |
| use_layerwise | 内容缺失，需要人工补齐。 | false | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 须与 `kv_connector_extra_config.use_layerwise` 一致。 |

### ASCEND_GLOBAL_RESOURCE_CONFIG 字段

| 参数 | 类型 | 默认值 | 必填 | 取值范围 | 说明 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| comm_resource_config.protocol_desc | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | 如 `["hccs:device"]`、`["roce:device"]`、`["uboe:device"]` | MooncakeConnectorV1 PD 传输路径协议描述符。 |
| store.comm_resource_config.protocol_desc | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 否 | 如 `["roce:device"]` | AscendStoreConnector 使用的 Mooncake Store 流量协议描述符。 |
| comm_resource_config.listen_port | 内容缺失，需要人工补齐。 | 16666 | 否 | 内容缺失，需要人工补齐。 | 单边通信监听端口。独立 `mooncake_client` 需使用不同端口避免冲突。 |
| fabric_memory.max_capacity | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | 仅 fabric mem 不足时需要 | 整数（GB per process） | Fabric 内存配额。 |

### QoS 配置

| 参数 | 类型 | 默认值 | 必填 | 取值范围 | 说明 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| qos_priority | 内容缺失，需要人工补齐。 | 0 | 否 | [0, 4]（整数，越大优先级越高） | Mooncake 与 Memcache 后端均支持。无效值（非整数、超范围）会导致启动时快速失败。 |

QoS 可通过 `kv_connector_extra_config` 配置，自动注入后端配置：

```json
{
    "kv_connector": "AscendStoreConnector",
    "kv_role": "kv_both",
    "kv_connector_extra_config": {
        "qos_priority": 1,
        "lookup_rpc_port": "1",
        "backend": "mooncake",
        "use_layerwise": false
    }
}
```

`kv_connector_extra_config` 中的值优先于环境变量中已设置的相同参数；覆盖时 WARN 日志。

### 硬件相关环境变量

| 参数 | 类型 | 默认值 | 必填 | 取值范围 | 说明 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| ASCEND_GLOBAL_RESOURCE_CONFIG | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | Ascend 950 Products UBOE 时需要 | JSON 字符串 | 配置 UBOE 协议描述符，如 `{"comm_resource_config.protocol_desc":["uboe:device"]}`。 |
| ASCEND_LOCAL_COMM_RES | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | Ascend 950 Products UB 时需要 | JSON 字符串 | 如 `{"version":"1.3"}`。 |
| ASCEND_ENABLE_USE_FABRIC_MEM | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | A3 HCCS 推荐 | 1 / 0 | 启用统一内存地址直传方案。A3 系列推荐设为 1。 |
| HCCL_INTRA_ROCE_ENABLE | 内容缺失，需要人工补齐。 | 内容缺失，需要人工补齐。 | A2 ROCE 时需要 | 1 / 0 | A2 系列 ROCE 直传方案所需。 |

**硬件依赖快速参考：**

| 硬件系列 | HDK 要求 | CANN 要求 | 其他依赖 |
| :--- | :--- | :--- | :--- |
| 950PR/DT Ascend 950 Products | >= 25.6（搭配 mooncake >= v0.3.11） | >= 9.1.0 | 需挂载 UBOE/UB 相关设备与配置 |
| 800 I/T A3 | >= 26.0 或 >= 25.5（搭配 mooncake >= v0.3.11） | >= 9.0.0 | 灵衢计算网络 >= 1.5；推荐 `ASCEND_ENABLE_USE_FABRIC_MEM=1` |
| 800 I/T A2 | >= 25.5 推荐 | 内容缺失，需要人工补齐。 | `HCCL_INTRA_ROCE_ENABLE=1` 直传方案 |

## 调优建议（可选）

**依据原文上下文内容重组，请进行人工校验。**

### Mooncake SSD Offload 参数调优

- `MOONCAKE_OFFLOAD_LOCAL_BUFFER_SIZE_BYTES` 遇到 `BUFFER_OVERFLOW` 时增大，但不要高于 vLLM Worker 日志中 `Available KV cache memory` 值。必须使用字节字面量（如 `10737418240`），不支持 `10G`/`10GB` 格式。
- A3 + `ASCEND_ENABLE_USE_FABRIC_MEM=1` 下，`MOONCAKE_OFFLOAD_LOCAL_BUFFER_SIZE_BYTES` 必须对齐 1 GB（1073741824 的整数倍）。默认 1280 MB 未对齐，可能导致 `adxl MallocMem` 失败或 `FileStorage init` 段错误。
- Fabric mem 预算公式（每 rank）：
  ```text
  fabric_memory.max_capacity >= global_segment_size + MOONCAKE_OFFLOAD_LOCAL_BUFFER_SIZE_BYTES (+ 余量)
  ```
  若 quota 不足，部分 rank 在 `global_segment_size` 成功后 `MOONCAKE_OFFLOAD_LOCAL_BUFFER_SIZE_BYTES` 分配失败，报 `Memory_Allocation_Failure(EL0004)`。
- `MOONCAKE_OFFLOAD_TOTAL_SIZE_LIMIT_BYTES` 默认 2 TB 往往远超真实磁盘容量，务必显式设置为每 rank 的实际预算。例如 800 GB 磁盘、8 TP rank 的场景：
  ```shell
  export MOONCAKE_OFFLOAD_TOTAL_SIZE_LIMIT_BYTES=$((100 * 1024 * 1024 * 1024))
  export MOONCAKE_OFFLOAD_BUCKET_MAX_TOTAL_SIZE=$((100 * 1024 * 1024 * 1024))
  export MOONCAKE_OFFLOAD_BUCKET_EVICTION_POLICY=lru
  export MOONCAKE_OFFLOAD_LOCAL_BUFFER_SIZE_BYTES=1073741824   # 1 GB
  ```
- 主机内存预算：
  ```text
  host_memory_for_mooncake ~ TP x (global_segment_size + MOONCAKE_OFFLOAD_LOCAL_BUFFER_SIZE_BYTES + local_buffer_size)
  ```

### MooncakeStore 预热

启用了 `ASCEND_BUFFER_POOL` 的 MooncakeStore，建议在实际性能基准测试前执行预热。由于 HCCS 单边通信连接在实例启动后延迟创建，全连接需要一次性时间开销（4 MB 设备内存/连接）。预热建议：输入序列长度 8k、输出序列长度 1，请求总数 2-3x 设备数。

### Memcache UBS IO 内存池调优

- `ubsio.mem.size_in_gb` 上限公式：
  ```text
  maximum ubsio.mem.size_in_gb = min(3072, floor(available node memory for UBS IO (GB) / number of DRAM-enabled local services))
  ```
- 分离部署场景建议单进程 50 GB，其他场景建议 10 GB。
- 如需使用 L2.5 内存缓存能力，在上限范围内调大 `ubsio.mem.size_in_gb` 并配合调整 [ubsio.wcache.evict_water_level](https://gitcode.com/Ascend/memcache/wiki/DRAM%20+%20SSD%20%E5%A4%9A%E7%BA%A7%E6%B1%A0%E5%8C%96%E9%85%8D%E7%BD%AE%E6%8C%87%E5%8D%97.md#ubsiowcacheevict_water_level)。

### Yuanrong Worker 参数调优

- `rpc_thread_num`、`oc_thread_num` 等线程计数为调优起点，需根据可用 CPU 核心数和请求吞吐量调整。
- `shared_memory_size_mb=40960` 时至少预留 20480 个 2 MiB 大页：
  ```bash
  grep -E "HugePages_Total|HugePages_Free|Hugepagesize" /proc/meminfo
  ```
- Worker `-w` 会消耗后续命令行参数，所有 `dscli start` 选项（如 `--timeout`）须放在 `-w` 之前。

### QoS 优先级

- `qos_priority` 取值范围 [0, 4]（整数），0 为默认。值越大优先级越高。
- 通过 `kv_connector_extra_config` 配置。在 Mooncake 后端中，`qos_priority` 会合并到已有的 `ASCEND_GLOBAL_RESOURCE_CONFIG` 中（保留其他字段）。当 `ASCEND_GLOBAL_RESOURCE_CONFIG` 未设置时，配置 `qos_priority` 也会创建该配置。

## 常见问题（可选）

**依据原文上下文内容重组，请进行人工校验。**

公共 FAQ 引用：
- [Mooncake Store Deployment Guide](https://github.com/kvcache-ai/Mooncake/blob/main/docs/source/deployment/mooncake-store-deployment-guide.md)
- [SSD Offload](https://github.com/kvcache-ai/Mooncake/blob/main/docs/source/deployment/ssd/ssd-offload.md)
- [HIXL 常见问题定位手册](https://gitcode.com/cann/hixl/wiki/HIXL%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98%E5%AE%9A%E4%BD%8D%E6%89%8B%E5%86%8C.md)
- [Memcache FAQ](https://gitcode.com/Ascend/memcache/wiki/FAQ.md)

### 问题 1：failed to put/get key

**问题描述：** vLLM 上报 failed `put` 或 `get` 操作。

**原因分析：** 需先判断错误是否由 Mooncake 自身报告：
- `put` 失败：Mooncake 日志中出现 `NO_AVAILABLE_HANDLE` 或 `BatchPut failed ... due to insufficient space`，通常是 eviction 后剩余空间不足以容纳一次 `BatchPut` 请求。
- `get` 失败：Mooncake 日志中出现 `lease_expired_before_data_transfer_completed key=...` 或返回 `LEASE_EXPIRED`，说明 KV 对象租约在数据传输完成前过期。

**解决步骤：**
1. 判断错误来源。若为 Mooncake 报告的 `put` 失败，确保 eviction 策略剩余空间（如 `1 - eviction_ratio`）可容纳一次 batch put，或增大可用容量、增加 eviction headroom、减小 batch size。
2. 若为 `get` 失败，增大 `mooncake_master` 的 `--default_kv_lease_ttl`，并保持其大于 `ASCEND_CONNECT_TIMEOUT` 与 `ASCEND_TRANSFER_TIMEOUT`。
3. 若错误并非 Mooncake 报告，则疑似 HIXL（ascend_direct）传输层问题，收集 `/root/ascend/log/debug/plog` 下的 plog 文件排查。

### 问题 2：SEGMENT_NOT_FOUND（SSD Offload）

**问题描述：** 客户端日志出现 `OffloadObjectHeartbeat failed, error code is SEGMENT_NOT_FOUND`，该 rank 的 SSD Offload 停止，直到 segment 重新注册。

**原因分析：** Master 已卸载该 rank 的 `LOCAL_DISK` segment（通常发生在 Ping 停止刷新 TTL 触发 `client_expired` 之后）。典型触发场景为 `enable_cpu_binding=true`：Mooncake 初始化时启动 Ping，之后 vLLM-Ascend `bind_cpus()` 执行 `migratepages`/IRQ 绑定，Ping 线程未绑定 CPU，可能在默认 `client_ttl=10` 内丢心跳。

**解决步骤：**
1. 临时方案：调大 Master TTL，例如 `mooncake_master ... --client_ttl=120`，按初始化/预热窗口调整（通常 60-120 足够）。
2. 恢复方案：升级 Mooncake 到 > v0.3.11（main branch），可自动重挂载 `LOCAL_DISK` 并重扫元数据。
3. 根治方案：将存储 Ping 线程绑定到 release/isolated CPU（Mooncake 侧修改）。
4. 调试重启时，将 Master 与 vLLM 一起重启，避免遗留 `segment_already_exists` 状态。

### 问题 3：Fabric memory 未对齐导致分配失败

**问题描述：** `adxl MallocMem` / `aclrtMapMem` 报 `Invalid_Argument`；SSD Offload 开启时 `MOONCAKE_OFFLOAD_LOCAL_BUFFER_SIZE_BYTES` 分配失败可能导致 `FileStorage init` 段错误并中止 vLLM 启动。

**原因分析：** A3 + `ASCEND_ENABLE_USE_FABRIC_MEM=1` 下，每项 fabric mem 分配必须为 1 GB 整数倍，Mooncake 不会自动向上取整。默认 1280 MB（1.25 GB）未对齐。

**解决步骤：**
1. 将 `MOONCAKE_OFFLOAD_LOCAL_BUFFER_SIZE_BYTES` 设置为 1 GB 整数倍，如 `1073741824`（1 GB）。
2. 按公式配置 fabric mem quota：`fabric_memory.max_capacity >= global_segment_size + MOONCAKE_OFFLOAD_LOCAL_BUFFER_SIZE_BYTES (+ 余量)`，例如：
   ```bash
   export ASCEND_ENABLE_USE_FABRIC_MEM=1
   export MOONCAKE_OFFLOAD_LOCAL_BUFFER_SIZE_BYTES=1073741824   # 1 GB, fabric-mem aligned
   export ASCEND_GLOBAL_RESOURCE_CONFIG='{"fabric_memory.max_capacity":32}'
   ```
3. 避免使用 `1280MB`、`512MB`、`1.5GB` 等未对齐值。注意 `mooncake.json` 中的 `local_buffer_size` 在 fabric mem 模式下不被使用。

### 问题 4：MOONCAKE_OFFLOAD_LOCAL_BUFFER_SIZE_BYTES 过小（BUFFER_OVERFLOW）

**问题描述：** SSD 读取失败，报 `BUFFER_OVERFLOW`（`error_code=-10`）于 `FileStorage::AllocateBatch`，且 `kv_load_failure_policy=fail` 时 vLLM 可能失败。

**原因分析：** `enable_ssd_offload=true` 时 Mooncake 分配独立的 per-rank SSD 读写缓冲区，该缓冲区独立于 `mooncake.json` 的 `global_segment_size`，增大 segment 不能修复 `BUFFER_OVERFLOW`。

**解决步骤：**
1. 增大 `MOONCAKE_OFFLOAD_LOCAL_BUFFER_SIZE_BYTES`，但不高于 vLLM Worker 日志 `Available KV cache memory` 值：
   ```text
   (Worker_TP0_EP0 pid=21240) INFO ... Available KV cache memory: XX
   ```
2. 示例：`export MOONCAKE_OFFLOAD_LOCAL_BUFFER_SIZE_BYTES=10737418240`（10 GB）。
3. 仅使用字节字面量；`10G`/`10GB` 会被忽略并回退到 1280 MB 默认值。
4. 调优后验证：启动时每个 rank 打印 `AlignedClientBufferAllocator: allocated <N> bytes`；负载下无 `BUFFER_OVERFLOW` / `Failed to get ... keys out of ... error_codes=[-10]`。
5. 若大缓冲区下仍失败，检查重叠加载（`load_async`）。

### 问题 5：Memcache 相关问题

**内容缺失，需要人工补齐。** Memcache 的故障排查请参考 [Memcache FAQ](https://gitcode.com/Ascend/memcache/wiki/FAQ.md)。

### 问题 6：DSv4 已知问题（临时）

**内容缺失，需要人工补齐。** DSv4 临时已知问题请参考 [vllm-ascend issue #9975](https://github.com/vllm-project/vllm-ascend/issues/9975)。