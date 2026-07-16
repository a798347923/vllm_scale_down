# vLLM Scale-Down：面向 Ascend 910C 的弹性容错方案

[English](README.md)

---

## 概述

**vLLM Scale-Down** 为运行在华为昇腾 910C NPU 上的 [vLLM](https://github.com/vllm-project/vllm) 提供弹性容错能力。它使 vLLM 能够**自动检测 NPU 硬件故障、暂停受影响的 DP rank、执行缩容、并持续提供推理服务**——无需重启服务。

当 NPU 卡掉线或变得不健康时，系统会：

1. **检测**故障（通过硬件轮询或引擎崩溃上报）
2. **暂停**受影响的 DP rank，停止接收新请求
3. **缩容**——重新分配专家、重载权重、重建通信组
4. **恢复**在剩余健康 NPU 上继续提供服务

效果：您的 vLLM 部署能够**容忍 NPU 故障**，以降低后的容量持续服务，而不是整体崩溃。

---

## 核心特性

| 特性 | 说明 |
|------|------|
| **硬件故障检测** | 通过 DCMI 接口轮询 NPU 健康状态，检测掉卡和错误码 |
| **引擎级故障上报** | 引擎崩溃通过 ZMQ 在秒级内捕获并上报 |
| **优雅缩容** | 暂停受影响 rank，重新分配专家，重载权重——无需重启 |
| **动态 EPLB 集成** | 故障后通过 EPLB（专家并行负载均衡）框架重新平衡专家放置 |
| **外部 API 控制** | REST API（`/fault_tolerance/apply`）支持 pause、retry、scale_down 指令 |
| **零停机恢复** | 重新配置后，存活的 NPU 自动恢复服务 |

---

## 快速上手

### 前置条件

- 华为昇腾 910C
- 已安装 vLLM v0.18.0 及 Ascend 后端（`vllm_ascend`）
- DCMI 库（`/usr/local/dcmi/libdcmi.so`，可选——仅启动 NPU 监控程序时需要）
- Python 依赖：`zmq`、`msgspec`、`requests`

### Step 0：拉取官方 Docker 镜像

拉取 vllm-ascend 官方镜像，确保 CANN、torch_npu 等依赖版本正确：

```bash
docker pull quay.io/ascend/vllm-ascend:v0.18.0-a3
```

然后启动并进入容器（后续所有步骤均在容器内执行）：

```bash
docker run -it --net=host --ipc=host --privileged=true -v /usr/local/sbin/npu-smi:/usr/local/sbin/npu-smi -v /etc/ascend_install.info:/etc/ascend_install.info -v /usr/local/Ascend/driver:/usr/local/Ascend/driver -v /usr/local/dcmi:/usr/local/dcmi quay.io/ascend/vllm-ascend:v0.18.0-a3 bash
```

### Step 1: 打补丁

Docker 镜像中已包含 vLLM 和 vLLM-Ascend 源码，位于 `/vllm-workspace/` 目录下。检查分支和 commit 是否正确，确认后打补丁：

```bash
# 检查 vLLM 分支和 commit
cd /vllm-workspace/vllm
git fetch --all
git checkout v0.18.0
git reset --hard bcf2be96
git apply /path/to/patches/vllm_scale_down.patch

# 检查 vLLM-Ascend 分支和 commit
cd /vllm-workspace/vllm-ascend
git fetch --all
git checkout v0.18.0
git reset --hard 4a533861
git apply /path/to/patches/vllm_ascend_scale_down.patch
```

### Step 2: 安装

```bash
# 安装 vLLM
cd /vllm-workspace/vllm
VLLM_TARGET_DEVICE=empty pip install -e .

# 安装 vLLM Ascend
cd /vllm-workspace/vllm-ascend
git submodule update --init --recursive
pip install -e .
```

### Step 3: 启动 vLLM 服务

```bash
# 终端 1：启用容错启动 vLLM
cd vllm-ascend
bash examples/Fault-Tolerance-scale/serve_qwen.sh \
    --dp 4 \
    --re 48 \
    --fault-port 22867 \
    --recovery-timeout 120 \
    --port 8006
```

关键参数说明：

| 参数 | 作用 |
|------|------|
| `--enable-fault-tolerance` | 开启容错框架 |
| `--fault-tolerance-config '{"external_fault_notify_port":22867}'` | 容错配置：设置健康状态广播端口（ZMQ PUB） |
| `--data-parallel-size 4` | DP rank 数量（每个 rank 使用一张 NPU） |
| `--gloo-timeout-seconds 15` | Gloo 进程组超时时间（秒） |
| `--recovery-timeout 120` | 容错超时时间——等待容错指令的最长时间（秒），超时后重新抛出原始错误 |
| `--additional-config '{"eplb_config":...}'` | 启用动态 EPLB 进行专家重分配 |

### Step 4（可选）：启动监控

```bash
# 终端 2：启动故障监控
python examples/Fault-Tolerance-scale/scale_down.py \
    --npu-ids 0,1,2,3 \
    --interval-time 3 \
    --external-fault-notify-port 22867 \
    --port 8006
```

监控程序是**可选的**。它通过 DCMI 轮询 NPU 硬件健康状态，当检测到掉卡时自动发送 `pause` + `scale_down` 指令——仅适用于 **NPU 硬件故障**场景。

**不启动监控时**，容错框架同样生效。当引擎 busy loop 中发生任何异常时，框架会：

1. **拦截异常**，保持服务不退出（在 `engine_recovery_timeout_sec` 时间窗口内）
2. **自动暂停**受影响的 DP rank，停止接收新请求
3. 等待用户通过 REST API 手动发送恢复指令

可以通过 `GET /fault_tolerance/status` 查看当前引擎健康状态，再通过 `POST /fault_tolerance/apply` 发送 `retry` 或 `scale_down`：

```bash
# 查看引擎状态
curl http://localhost:8006/fault_tolerance/status

# 发送 retry 恢复所有暂停的引擎
curl -X POST http://localhost:8006/fault_tolerance/apply \
    -H "Content-Type: application/json" \
    -d '{"instruction":"retry","params":{"timeout":30}}'

# 或发送 scale_down 移除 DP rank 2
curl -X POST http://localhost:8006/fault_tolerance/apply \
    -H "Content-Type: application/json" \
    -d '{"instruction":"scale_down","params":{"timeout":30,"exclude_dp_ranks":[2]}}'
```

---

## 架构

### 组件总览

```
                          External Monitor
                          (scale_down.py)
                                |
                    +-----------+-----------+
                    |                       |
          ZMQ SUB (fault)        HTTP POST /fault_tolerance/apply
                    |                       |
                    v                       v
            +-------+-------+     +--------+--------+
            |  API Server   |     |  API Server     |
            |  (FastAPI)    |     |  (FastAPI)      |
            +-------+-------+     +--------+--------+
                    |                       |
                    v                       v
            +-------+-----------------------+--------+
            |          ClientSentinel                 |
            |  (one per vLLM instance)                |
            |  - Receives fault reports via ZMQ       |
            |  - Publishes engine health status       |
            |  - Dispatches pause/retry/scale_down    |
            +--+------------+------------+------------+
               |            |            |
      +--------+--+  +-----+------+  +--+--------+
      | EngineCore |  | EngineCore |  | EngineCore|
      | Sentinel   |  | Sentinel   |  | Sentinel  |
      | (DP rank 0)|  | (DP rank 1)|  | (DP rank 2)|
      +-----+------+  +-----+------+  +-----+-----+
            |                |               |
      +-----+----------------+---------------+-----+
      |           EngineCore (busy loop)            |
      |  wrapped with @fault_tolerant_wrapper       |
      +-----+----------------+---------------+-----+
            |                |               |
      +-----+------+  +-----+------+  +-----+------+
      |   Worker   |  |   Worker   |  |   Worker   |
      |  Sentinel  |  |  Sentinel  |  |  Sentinel  |
      |   (NPU)    |  |   (NPU)    |  |   (NPU)    |
      +------------+  +------------+  +------------+
```

### 容错工作流程

```
    NPU Card Drop / Engine Crash
                |
                v
    +-----------+-----------+
    | 1. DETECTION          |
    |   - DCMI polls health |
    |   - Engine reports    |
    |     via ZMQ           |
    +-----------+-----------+
                |
                v
    +-----------+-----------+
    | 2. PAUSE              |
    |   - Stop affected DP  |
    |     ranks from        |
    |     processing        |
    |   - Preempt in-flight |
    |     requests          |
    +-----------+-----------+
                |
                v
    +-----------+-----------+
    | 3. SCALE DOWN         |
    |   - Compute new rank  |
    |     mapping           |
    |   - Redistribute      |
    |     experts (EPLB)    |
    |   - Reload weights    |
    |     CPU -> NPU        |
    |   - Reinit comm       |
    |     groups (Gloo)     |
    +-----------+-----------+
                |
                v
    +-----------+-----------+
    | 4. RESUME             |
    |   - Remaining NPUs    |
    |     continue serving  |
    |   - Health status     |
    |     published         |
    +-----------+-----------+
```

### 通信通道

| 通道 | 协议 | 方向 | 用途 |
|------|------|------|------|
| Engine Fault Socket | ZMQ DEALER/ROUTER | Engine -> ClientSentinel | 上报引擎异常 |
| Fault State PUB/SUB | ZMQ PUB/SUB | ClientSentinel -> 外部 | 广播引擎健康状态 |
| FT Request/Result | ZMQ DEALER/PUSH | ClientSentinel -> Engine | 下发 pause/retry/scale_down |
| Worker Command | ZMQ ROUTER/DEALER | EngineCore -> Worker | Worker 级别控制 |
| HTTP API | REST | 外部 -> API Server | 外部容错控制 |

---

## 配置参数

### 命令行参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--enable-fault-tolerance` | `False` | 启用容错框架 |
| `--fault-tolerance-config` | `None` | 容错配置 JSON 字典（自动启用 FT）。支持 `engine_recovery_timeout_sec`、`external_fault_notify_port` 等字段 |
| `--gloo-timeout-seconds` | `None`（回退到 600） | Gloo 进程组超时时间 |

### 容错配置

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `engine_recovery_timeout_sec` | `120` | 等待恢复指令的超时时间（秒），超时后重新抛出原始错误 |
| `enable_fault_tolerance_rebalance` | `False` | 故障后重新调用 EPLB 进行专家负载均衡 |
| `internal_fault_report_port` | `22866` | 引擎向 ClientSentinel 上报故障的端口（内部） |
| `external_fault_notify_port` | `22867` | ClientSentinel 向外部发布故障通知的端口（外部 ZMQ PUB） |

---

## REST 接口

### POST /fault_tolerance/apply

向运行中的 vLLM 实例发送容错指令。

**请求体：**

```json
{
    "fault_tolerance_instruction": "pause | retry | scale_down",
    "fault_tolerance_timeout": 30,
    "fault_tolerance_params": {}
}
```

**指令：**

| 指令 | 参数 | 说明 |
|------|------|------|
| `pause` | `exclude_dp_ranks`（可选） | 暂停指定或所有 DP rank 的请求处理 |
| `retry` | — | 重试：清理 worker 状态并重新初始化通信 |
| `scale_down` | `exclude_dp_ranks`（必填） | 移除指定 DP rank 并重新分配专家 |

### GET /fault_tolerance/status

返回当前引擎健康状态。

**响应：**

```json
{
    "total_engines": 4,
    "engines": [
        {"id": 0, "status": "healthy"},
        {"id": 1, "status": "healthy"},
        {"id": 2, "status": "dead"},
        {"id": 3, "status": "healthy"}
    ]
}
```

---

## 完整示例

**场景：8 卡部署，容忍单卡故障**

```bash
# 终端 1：启动 vLLM 服务
bash examples/Fault-Tolerance-scale/serve_qwen.sh \
    --dp 8 \
    --re 48 \
    --fault-port 22876 \
    --recovery-timeout 180

# 终端 2：启动故障监控
python examples/Fault-Tolerance-scale/scale_down.py \
    --npu-ids 0,1,2,3,4,5,6,7 \
    --interval-time 3 \
    --external-fault-notify-port 22876
```

当 NPU #3 发生故障时：
- 监控程序通过 DCMI 轮询检测到故障
- 发送 `pause` 指令冻结 DP rank 3
- 发送 `scale_down` 指令移除 DP rank 3
- vLLM 在剩余 7 张 NPU 上重新分配专家
- 服务以 7/8 的容量继续运行

---

## 限制

| 限制 | 说明 |
|------|------|
| 仅支持 Ascend 910C | 当前仅支持华为昇腾 910C NPU |
| 仅支持 TP = 1 | 缩容当前仅支持 tensor parallel size 为 1 |
| 不支持扩容 | 缩容后无法添加 NPU（单向缩容） |
| 不支持 Pipeline Parallel | PP 与当前缩容实现不兼容 |
| 冗余专家数量要求（缩容限制） | 执行缩容时，健康卡上的冗余专家总数必须大于故障卡上的非冗余专家数量，否则缩容后故障卡上的专家无法被完全覆盖 |

---

## 常见问题

**Q: vLLM 启动失败，提示 "DCMI library not found"**
设置 `DCMI_LIB_PATH` 环境变量：
```bash
export DCMI_LIB_PATH=/path/to/libdcmi.so
```

**Q: 缩容过程中权重重载卡住**
确保 `engine_recovery_timeout_sec` 大于 Gloo 超时时间。默认值为 120s；如果模型较大，可能需要增加此值。

**Q: 如何验证容错功能是否正常工作？**
检查 `/fault_tolerance/status` 接口：
```bash
curl http://localhost:8006/fault_tolerance/status
```

---

## 补丁

本仓库提供两个补丁：

| 补丁 | 目标仓库 | 说明 |
|------|----------|------|
| `patches/vllm_scale_down.patch` | `vllm-project/vllm`（v0.18.0） | 核心容错框架：sentinel 层级、ZMQ 通信、HTTP API、引擎健康监控、缩容编排 |
| `patches/vllm_ascend_scale_down.patch` | `vllm-project/vllm-ascend`（v0.18.0） | Ascend 适配：NPU worker sentinel、ScaleDownHelper（7 阶段工作流）、DCMI 硬件监控、EPLB 故障重排策略 |

---

## 贡献

欢迎贡献！请提交 issue 或 pull request。

---

## 许可证

Apache License 2.0
