# Release Notes

## v0.1.0

- **Version:** v0.1.0
- **Release Date:** 2026-07-16
- **Releaser:** a798347923

### Features

- Core fault tolerance framework with three-level sentinel hierarchy (ClientSentinel, EngineCoreSentinel, WorkerSentinel)
- ZMQ-based communication for fault reporting and instruction dispatch
- REST API for external fault tolerance control (`/fault_tolerance/apply`, `/fault_tolerance/status`)
- Graceful scale-down: pause affected ranks, redistribute experts, reload weights, reinitialize communication groups
- Dynamic EPLB integration for expert load balancing after fault
- External NPU hardware monitor via DCMI polling
- Support for Qwen3-30B-A3B model with W8A8 quantization on Ascend 910C

### Patches

- `patches/vllm_scale_down.patch`: Core fault tolerance framework for vLLM v0.18.0
- `patches/vllm_ascend_scale_down.patch`: Ascend-specific adaptations for vllm-ascend v0.18.0
