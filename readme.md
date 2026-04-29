# GLM 5.1 FP8 — IBM Cloud H200 Setup Guide

**VSI Profile:** `gx3d-160x1792x8h200`
**Image:** `ibm-redhat-ai-nvidia-1-5-2-amd64-1` (NVIDIA drivers + Podman pre-installed)

---

## Folder Structure on Block Volume

```
/mnt/data/
├── miniconda/              ← Miniconda install
├── conda-envs/
│   └── llm/               ← Conda environment (includes podman-compose)
├── models/
│   └── glm-5.1-fp8/       ← GLM 5.1 FP8 weights (~756GB)
├── images/                ← Saved container images (.tar)
│   ├── vllm-openai.tar
│   ├── litellm.tar
│   └── open-webui.tar
├── scripts/               ← Stack config files
│   ├── docker-compose.yml
│   ├── litellm_config.yaml
│   └── .env
└── hf-cache/              ← HuggingFace download cache
```

---

## FIRST TIME SETUP

### 1. Mount Block Storage Volume

```bash
# Find your volume (look for the large disk e.g. 2.9T)
lsblk

# Format — first time only!
mkfs.ext4 /dev/vdd

# Mount
mkdir -p /mnt/data
mount /dev/vdd /mnt/data

# Persist across reboots
echo '/dev/vdd /mnt/data ext4 defaults 0 0' | sudo tee -a /etc/fstab

# Create folder structure
mkdir -p /mnt/data/miniconda
mkdir -p /mnt/data/conda-envs
mkdir -p /mnt/data/models
mkdir -p /mnt/data/images
mkdir -p /mnt/data/scripts
mkdir -p /mnt/data/hf-cache
```

> Note: On RHEL AI (ostree), `/mnt` is a symlink to `/var/mnt` — both paths work.

---

### 2. Install Miniconda (Once — Stored on Volume)

Install to `/mnt/data/miniconda` so it survives VSI rebuilds.

```bash
curl -O https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh -b -p /mnt/data/miniconda

# Add to PATH
echo 'export PATH=/mnt/data/miniconda/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# Reject anaconda ToS — use conda-forge only
conda config --add channels conda-forge
conda config --set channel_priority strict
conda config --remove channels defaults
```

---

### 3. Create Conda Environment (Once — Stored on Volume)

All tools including `podman-compose` are installed in the conda env.
Activating the env makes everything available — no system-wide copies needed.

```bash
conda create --prefix /mnt/data/conda-envs/llm python=3.11 -c conda-forge --override-channels
conda activate /mnt/data/conda-envs/llm

# Install all tools inside the env
pip install podman-compose
pip install "huggingface_hub[cli]"

# Verify
podman-compose --version
huggingface-cli --version

# Auto-activate on login
echo 'conda activate /mnt/data/conda-envs/llm' >> ~/.bashrc
```

---

### 4. Configure NVIDIA CDI for Podman (Once per VSI)

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml

# Verify all 8x H200 visible
sudo podman run --rm --device nvidia.com/gpu=all \
  docker.io/nvidia/cuda:12.3.1-base-ubuntu22.04 nvidia-smi
```

---

### 5. Pull and Save Container Images (Once — Stored on Volume)

Pull images and save to `/mnt/data/images/` so they never need to be re-pulled:

```bash
# Pull
sudo podman pull docker.io/vllm/vllm-openai:latest
sudo podman pull ghcr.io/berriai/litellm:main-latest
sudo podman pull ghcr.io/open-webui/open-webui:main

# Save to volume
sudo podman save docker.io/vllm/vllm-openai:latest \
  -o /mnt/data/images/vllm-openai.tar

sudo podman save ghcr.io/berriai/litellm:main-latest \
  -o /mnt/data/images/litellm.tar

sudo podman save ghcr.io/open-webui/open-webui:main \
  -o /mnt/data/images/open-webui.tar
```

---

### 6. Download GLM 5.1 FP8 (~756GB — Once — Stored on Volume)

Use tmux to keep download running in background:

```bash
# Start tmux session
tmux new -s glm-download

# Run download inside tmux
huggingface-cli download zai-org/GLM-5.1-FP8 \
  --local-dir /mnt/data/models/glm-5.1-fp8 \
  --local-dir-use-symlinks False \
  --cache-dir /mnt/data/hf-cache

# Detach (download keeps running): Ctrl+B then D
# Reattach to check progress:      tmux attach -t glm-download
```

Monitor progress:
```bash
watch -n 30 du -sh /mnt/data/models/glm-5.1-fp8
```

---

## ON A NEW VSI (Volume Already Has Everything)

When you create a new VSI and attach the same volume, you only need to:

### 1. Mount the Volume

```bash
mkdir -p /mnt/data
mount /dev/vdd /mnt/data
echo '/dev/vdd /mnt/data ext4 defaults 0 0' | sudo tee -a /etc/fstab
```

### 2. Restore PATH and Activate Conda

```bash
echo 'export PATH=/mnt/data/miniconda/bin:$PATH' >> ~/.bashrc
echo 'conda activate /mnt/data/conda-envs/llm' >> ~/.bashrc
source ~/.bashrc
```

Once activated, `podman-compose` and `huggingface-cli` are immediately available. ✅

### 3. Regenerate NVIDIA CDI (Required on Each New VSI)

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
```

### 4. Load Container Images from Volume

```bash
sudo podman load -i /mnt/data/images/vllm-openai.tar
sudo podman load -i /mnt/data/images/litellm.tar
sudo podman load -i /mnt/data/images/open-webui.tar

# Verify
sudo podman images
```

### 5. Start the Stack

```bash
cd /mnt/data/scripts
sudo podman-compose --project-directory /mnt/data/scripts up -d
```

---

## Verify Stack is Running

```bash
# Check all containers
sudo podman ps

# vLLM
curl http://localhost:8000/v1/models

# LiteLLM
curl http://localhost:4000/health/liveliness

# LiteLLM models (needs API key)
curl http://localhost:4000/v1/models \
  -H "Authorization: Bearer <your-LITELLM_MASTER_KEY>"
```

Open WebUI: `http://<vsi-ip>:3000`

---

## Useful Commands

```bash
# Check all 8x H200
nvidia-smi

# Check GPU inside vLLM container
sudo podman exec vllm-glm51 nvidia-smi

# Stop stack
sudo podman-compose --project-directory /mnt/data/scripts down

# Restart single service
sudo podman restart litellm

# Watch logs
sudo podman logs -f vllm-glm51
sudo podman logs -f litellm

# Check disk usage
df -h /mnt/data
du -sh /mnt/data/models/*
du -sh /mnt/data/images/*
```
