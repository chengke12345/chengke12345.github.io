compose.yml
```yaml
name: heteroserve

# 主备模型会复用 GPU 0、1，因此必须使用互斥 profile，不能同时启动。
x-vllm-common: &vllm-common
  build:
    context: .
    dockerfile: deploy/Dockerfile
    args:
      VLLM_VERSION: "${VLLM_VERSION:-v0.21.0}"
  image: "heteroserve/vllm-openai:${VLLM_VERSION:-v0.21.0}-sm75"
  init: true
  restart: unless-stopped
  ipc: host
  stop_grace_period: 2m
  environment:
    CUDA_DEVICE_ORDER: PCI_BUS_ID
    VLLM_LOGGING_LEVEL: "${VLLM_LOGGING_LEVEL:-INFO}"
    NCCL_DEBUG: "${NCCL_DEBUG:-WARN}"
    HF_HOME: /root/.cache/huggingface
    HF_HUB_OFFLINE: "${HF_HUB_OFFLINE:-1}"
    TRANSFORMERS_OFFLINE: "${TRANSFORMERS_OFFLINE:-1}"
    HF_TOKEN: "${HF_TOKEN:-}"
  volumes:
    - type: bind
      source: "${MODEL_ROOT:-/opt/models}"
      target: /models
      read_only: true
    - type: volume
      source: hf-cache
      target: /root/.cache/huggingface
  healthcheck:
    test:
      - CMD
      - python3
      - -c
      - "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=5)"
    interval: 30s
    timeout: 10s
    retries: 5
    start_period: 10m
  ulimits:
    memlock: -1
    stack: 67108864
  logging:
    driver: json-file
    options:
      max-size: 100m
      max-file: "3"

services:
  # 主模型：3 张 Turing/SM75 GPU，使用 PP=3。
  # Qwen3-32B-AWQ 为 FP16 activation + AWQ 4-bit weight。
  vllm-main:
    <<: *vllm-common
    profiles: [main]
    ports:
      - "${API_PORT:-8000}:8000"
    command:
      - "${MAIN_MODEL_PATH:-/models/Qwen3-32B-AWQ}"
      - --served-model-name
      - "${MAIN_SERVED_MODEL_NAME:-qwen3-32b-awq}"
      - --pipeline-parallel-size
      - "3"
      - --tensor-parallel-size
      - "1"
      - --dtype
      - float16
      - --quantization
      - awq_marlin
      - --gpu-memory-utilization
      - "${GPU_MEMORY_UTILIZATION:-0.88}"
      - --max-model-len
      - "${MAIN_MAX_MODEL_LEN:-8192}"
      - --max-num-seqs
      - "${MAIN_MAX_NUM_SEQS:-8}"
      - --enable-prefix-caching
      - --enable-chunked-prefill
      - --enable-reasoning
      - --reasoning-parser
      - deepseek_r1
      - --host
      - 0.0.0.0
      - --port
      - "8000"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ["0", "1", "2"]
              capabilities: [gpu]

  # 备用模型：2 张 Turing/SM75 GPU，使用 TP=2。
  # Qwen3-14B 权重原始 dtype 是 BF16，但 Turing 不支持原生 BF16，因此强制 FP16。
  vllm-backup:
    <<: *vllm-common
    profiles: [backup]
    ports:
      - "${API_PORT:-8000}:8000"
    command:
      - "${BACKUP_MODEL_PATH:-/models/Qwen3-14B}"
      - --served-model-name
      - "${BACKUP_SERVED_MODEL_NAME:-qwen3-14b}"
      - --pipeline-parallel-size
      - "1"
      - --tensor-parallel-size
      - "2"
      - --dtype
      - float16
      - --gpu-memory-utilization
      - "${GPU_MEMORY_UTILIZATION:-0.88}"
      - --max-model-len
      - "${BACKUP_MAX_MODEL_LEN:-8192}"
      - --max-num-seqs
      - "${BACKUP_MAX_NUM_SEQS:-8}"
      - --enable-prefix-caching
      - --enable-chunked-prefill
      - --enable-reasoning
      - --reasoning-parser
      - deepseek_r1
      - --host
      - 0.0.0.0
      - --port
      - "8000"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ["0", "1"]
              capabilities: [gpu]

volumes:
  hf-cache:

```