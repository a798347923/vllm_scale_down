# vLLM Scale-Down: Elastic Fault Tolerance for vLLM on Ascend 910C

[中文](README_ZH.md)

---

## Overview

**vLLM Scale-Down** brings elastic fault tolerance to [vLLM](https://github.com/vllm-project/vllm) on Huawei Ascend 910C NPU. It enables vLLM to **automatically detect NPU hardware failures, pause affected data-parallel ranks, scale them out, and keep serving** — all without restarting the service.

When a NPU card drops or becomes unhealthy, the system:

1. **Detects** the fault (via hardware polling or engine crash reporting)
2. **Pauses** the affected DP ranks to stop accepting new requests
3. **Scales down** by re-distributing experts, reloading weights, and reinitializing communication groups
4. **Resumes** serving on the remaining healthy NPUs

The result: your vLLM deployment **survives NPU failures** and continues serving with reduced capacity, instead of crashing entirely.

---

## Key Features

| Feature | Description |
|---------|-------------|
| **Hardware Fault Detection** | Polls NPU health via DCMI interface; detects card drops and error codes |
| **Engine-Level Fault Reporting** | Engine crashes are caught and reported via ZMQ within seconds |
| **Graceful Scale-Down** | Pauses affected ranks, redistributes experts, reloads weights — no restart needed |
| **Dynamic EPLB Integration** | Expert placement is re-balanced after fault via the EPLB (Expert Parallel Load Balancing) framework |
| **External API Control** | REST API (`/fault_tolerance/apply`) for pause, retry, and scale_down instructions |
| **Zero-Downtime Recovery** | Surviving NPUs resume serving automatically after reconfiguration |

---

## Quick Start

### Prerequisites

- Huawei Ascend 910C
- vLLM v0.18.0 installed with Ascend backend (`vllm_ascend`)
- DCMI library (`/usr/local/dcmi/libdcmi.so`, optional — only needed for the NPU monitor)
- Python packages: `zmq`, `msgspec`, `requests`

### Step 0: Pull the Official Docker Image

Pull the official vllm-ascend image to ensure CANN, torch_npu, and other dependencies are at the correct versions:

```bash
docker pull quay.io/ascend/vllm-ascend:v0.18.0-a3
```

Then start and enter the container (all following steps should be executed inside the container):

```bash
docker run -it --net=host --ipc=host --privileged=true -v /usr/local/sbin/npu-smi:/usr/local/sbin/npu-smi -v /etc/ascend_install.info:/etc/ascend_install.info -v /usr/local/Ascend/driver:/usr/local/Ascend/driver -v /usr/local/dcmi:/usr/local/dcmi quay.io/ascend/vllm-ascend:v0.18.0-a3 bash
```

### Step 1: Apply the Patches

The Docker image already contains vLLM and vLLM-Ascend source code under `/vllm-workspace/`. Check the branch and commit, and apply the patches:

```bash
# Check vLLM branch and commit
cd /vllm-workspace/vllm
git fetch --all
git checkout v0.18.0
git reset --hard bcf2be96
git apply /path/to/patches/vllm_scale_down.patch

# Check vLLM-Ascend branch and commit
cd /vllm-workspace/vllm-ascend
git fetch --all
git checkout v0.18.0
git reset --hard 4a533861
git apply /path/to/patches/vllm_ascend_scale_down.patch
```

### Step 2: Install

```bash
# Install vLLM
cd /vllm-workspace/vllm
VLLM_TARGET_DEVICE=empty pip install -e .

# Install vLLM Ascend
cd /vllm-workspace/vllm-ascend
git submodule update --init --recursive
pip install -e .
```

### Step 3: Start vLLM Service

```bash
# Terminal 1: Start vLLM with fault tolerance enabled
cd vllm-ascend
bash examples/Fault-Tolerance-scale/serve_qwen.sh \
    --dp 4 \
    --re 48 \
    --fault-port 22867 \
    --recovery-timeout 120 \
    --port 8006
```

Key flags explained:

| Flag | Purpose |
|------|---------|
| `--enable-fault-tolerance` | Turns on the fault tolerance framework |
| `--fault-tolerance-config '{"external_fault_notify_port":22867}'` | FT config: sets the health broadcast port (ZMQ PUB) |
| `--data-parallel-size 4` | Number of DP ranks (each uses one NPU) |
| `--gloo-timeout-seconds 15` | Gloo process group timeout in seconds |
| `--recovery-timeout 120` | Engine recovery timeout — how long to wait for FT instructions before re-raising the error |
| `--additional-config '{"eplb_config":...}'` | Enables dynamic EPLB for expert redistribution |

### Step 4 (Optional): Start the Monitor

```bash
# Terminal 2: Start the fault monitor
python examples/Fault-Tolerance-scale/scale_down.py \
    --npu-ids 0,1,2,3 \
    --interval-time 3 \
    --port 8006
```

The monitor is **optional**. It uses DCMI to poll NPU hardware health and automatically sends `pause` + `scale_down` when a card failure is detected — only applicable to **NPU hardware faults**.

**Without the monitor**, the fault tolerance framework still works. When any exception occurs in the engine busy loop, the framework:

1. **Catches** the exception and keeps the service alive (for up to `engine_recovery_timeout_sec`)
2. **Auto-pauses** the affected DP ranks to stop processing new requests
3. Waits for you to manually send recovery instructions via the REST API

You can check engine health with `GET /fault_tolerance/status`, then send `retry` or `scale_down` via `POST /fault_tolerance/apply`:

```bash
# Check engine status
curl http://localhost:8006/fault_tolerance/status

# Send retry to recover all paused engines
curl -X POST http://localhost:8006/fault_tolerance/apply \
    -H "Content-Type: application/json" \
    -d '{"instruction":"retry","params":{"timeout":30}}'

# Or send scale_down to remove DP rank 2
curl -X POST http://localhost:8006/fault_tolerance/apply \
    -H "Content-Type: application/json" \
    -d '{"instruction":"scale_down","params":{"timeout":30,"exclude_dp_ranks":[2]}}'
```

---

## Architecture

### Component Overview

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

### Fault Tolerance Workflow

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

### Communication Channels

| Channel | Protocol | Direction | Purpose |
|---------|----------|-----------|---------|
| Engine Fault Socket | ZMQ DEALER/ROUTER | Engine -> ClientSentinel | Report engine exceptions |
| Fault State PUB/SUB | ZMQ PUB/SUB | ClientSentinel -> External | Broadcast engine health status |
| FT Request/Result | ZMQ DEALER/PUSH | ClientSentinel -> Engine | Dispatch pause/retry/scale_down |
| Worker Command | ZMQ ROUTER/DEALER | EngineCore -> Worker | Worker-level control |
| HTTP API | REST | External -> API Server | External fault tolerance control |

---

## Configuration

### CLI Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `--enable-fault-tolerance` | `False` | Enable the fault tolerance framework |
| `--fault-tolerance-config` | `None` | JSON dict for FT config (auto-enables FT). Accepts `engine_recovery_timeout_sec`, `external_fault_notify_port`, etc. |
| `--gloo-timeout-seconds` | `None` (falls back to 600) | Gloo process group timeout |
| `--recovery-timeout` | `120` | Shorthand for `engine_recovery_timeout_sec` — seconds to wait for FT instructions before re-raising the error |

### FaultToleranceConfig

| Field | Default | Description |
|-------|---------|-------------|
| `engine_recovery_timeout_sec` | `120` | Seconds to wait for recovery instructions before re-raising the original error |
| `enable_fault_tolerance_rebalance` | `False` | Re-invoke EPLB for expert load balancing after fault |
| `internal_fault_report_port` | `22866` | Port for engines to report faults to ClientSentinel (internal) |
| `external_fault_notify_port` | `22867` | Port for ClientSentinel to publish fault notifications (external ZMQ PUB) |

---

## REST API

### POST /fault_tolerance/apply

Send a fault tolerance instruction to the running vLLM instance.

**Request Body:**

```json
{
    "fault_tolerance_instruction": "pause | retry | scale_down",
    "fault_tolerance_timeout": 30,
    "fault_tolerance_params": {}
}
```

**Instructions:**

| Instruction | Params | Description |
|-------------|--------|-------------|
| `pause` | `exclude_dp_ranks` (optional) | Pause request processing on specified or all DP ranks |
| `retry` | — | Retry: clean worker states and reinitialize communication |
| `scale_down` | `exclude_dp_ranks` (required) | Remove specified DP ranks and redistribute experts |

### GET /fault_tolerance/status

Returns current engine health status.

**Response:**

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

## Complete Example

**Scenario: 8-NPU deployment, tolerating single NPU failure**

```bash
# Terminal 1: Start vLLM service
bash examples/Fault-Tolerance-scale/serve_qwen.sh \
    --dp 8 \
    --re 48 \
    --fault-port 22876 \
    --recovery-timeout 180

# Terminal 2: Start fault monitor
python examples/Fault-Tolerance-scale/scale_down.py \
    --npu-ids 0,1,2,3,4,5,6,7 \
    --interval-time 3 \
    --external-fault-notify-port 22876
```

When NPU #3 fails:
- Monitor detects the fault via DCMI polling
- Sends `pause` instruction to freeze DP rank 3
- Sends `scale_down` instruction to remove DP rank 3
- vLLM redistributes experts across remaining 7 NPUs
- Service continues with 7/8 capacity

---

## Limitations

| Limitation | Description |
|------------|-------------|
| Ascend 910C only | Currently only supports Huawei Ascend 910C NPU |
| TP = 1 only | Scale-down currently only supports tensor-parallel size of 1 |
| No scale-up | Cannot add NPU capacity back after scale-down (one-directional) |
| Pipeline parallel unsupported | PP is not compatible with the current scale-down implementation |
| Redundant experts budget (scale-down) | To perform scale-down successfully, total redundant experts across healthy cards must exceed the non-redundant experts on the failed card; otherwise the failed card's experts cannot be fully covered after removal |

---

## Troubleshooting

**Q: vLLM fails to start with "DCMI library not found"**
Set the `DCMI_LIB_PATH` environment variable:
```bash
export DCMI_LIB_PATH=/path/to/libdcmi.so
```

**Q: Scale-down hangs during weight reload**
Ensure `engine_recovery_timeout_sec` is larger than your Gloo timeout. The default is 120s; if you have a large model, you may need to increase it.

**Q: How do I verify fault tolerance is working?**
Check the `/fault_tolerance/status` endpoint:
```bash
curl http://localhost:8006/fault_tolerance/status
```

---

## Patches

This repository provides two patches:

| Patch | Target | Description |
|-------|--------|-------------|
| `patches/vllm_scale_down.patch` | `vllm-project/vllm` (v0.18.0) | Core fault tolerance framework: sentinel hierarchy, ZMQ communication, HTTP API, engine health monitoring, scale-down orchestration |
| `patches/vllm_ascend_scale_down.patch` | `vllm-project/vllm-ascend` (v0.18.0) | Ascend-specific: NPU worker sentinel, ScaleDownHelper (7-phase workflow), DCMI hardware monitoring, EPLB fault rearrangement policy |

---

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

---

## License

Apache License 2.0
