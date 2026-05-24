# Linode GPU Instances

> High-performance GPU compute for ML training, rendering, and scientific computing.

## Overview

Linode GPU instances provide dedicated NVIDIA GPUs for machine learning, deep learning, video processing, and scientific computing. Each GPU instance includes dedicated GPU cards, high-performance CPUs, and NVMe storage.

## Key Concepts

| Term | Definition |
|------|------------|
| GPU Plan | Instance type with dedicated GPU |
| CUDA | NVIDIA parallel computing platform |
| vGPU | Not supported (dedicated GPUs only) |
| GPU RAM | Video memory on GPU card |
| Tensor Cores | Specialized cores for ML |

## GPU Plans

| Plan | GPU | GPU RAM | vCPU | RAM | Storage | Price/mo |
|------|-----|---------|------|-----|---------|----------|
| GPU RTX 4000 | 1× RTX 4000 | 8 GB | 8 | 32 GB | 640 GB | $650 |
| GPU 2× RTX 4000 | 2× RTX 4000 | 16 GB | 16 | 64 GB | 1.28 TB | $1,300 |
| GPU RTX 6000 | 1× RTX 6000 | 24 GB | 8 | 32 GB | 640 GB | $1,000 |
| GPU 2× RTX 6000 | 2× RTX 6000 | 48 GB | 16 | 64 GB | 1.28 TB | $2,000 |

## GPU Specifications

### NVIDIA Quadro RTX 4000

| Spec | Value |
|------|-------|
| CUDA Cores | 2,304 |
| Tensor Cores | 288 |
| GPU Memory | 8 GB GDDR6 |
| Memory Bandwidth | 416 GB/s |
| FP32 Performance | 7.1 TFLOPS |
| RT Cores | 36 |

### NVIDIA Quadro RTX 6000

| Spec | Value |
|------|-------|
| CUDA Cores | 4,608 |
| Tensor Cores | 576 |
| GPU Memory | 24 GB GDDR6 |
| Memory Bandwidth | 672 GB/s |
| FP32 Performance | 16.3 TFLOPS |
| RT Cores | 72 |

## Architecture

```
+---------------------------------------------------------------+
|                      GPU Linode Instance                      |
|                                                               |
|  +-------------------------+  +---------------------------+   |
|  |         CPU             |  |          GPU              |   |
|  |   8/16 vCPU Cores       |  |   NVIDIA Quadro RTX       |   |
|  |   AMD EPYC              |  |   8-24 GB VRAM            |   |
|  +-------------------------+  +---------------------------+   |
|                                                               |
|  +-------------------------+  +---------------------------+   |
|  |       System RAM        |  |       NVMe Storage        |   |
|  |       32-64 GB          |  |       640 GB - 1.28 TB    |   |
|  +-------------------------+  +---------------------------+   |
|                                                               |
|  PCIe 4.0 x16 connection to GPU                               |
+---------------------------------------------------------------+
```

## Create GPU Instance

### CLI

```bash
# Create GPU instance
linode-cli linodes create \
  --type g1-gpu-rtx6000-1 \
  --region us-east \
  --image linode/ubuntu22.04 \
  --root_pass "SecurePassword!" \
  --label gpu-ml-1

# List GPU plans
linode-cli linodes types --text | grep gpu
```

### Terraform

```hcl
resource "linode_instance" "gpu" {
  label      = "ml-training"
  type       = "g1-gpu-rtx6000-1"
  region     = "us-east"
  image      = "linode/ubuntu22.04"

  authorized_keys = [var.ssh_public_key]
}
```

## Region Availability

GPU instances are available in select regions:

| Region | Availability |
|--------|--------------|
| Newark (us-east) | Yes |
| Fremont (us-west) | Yes |
| London (eu-west) | Limited |
| Frankfurt (eu-central) | Limited |

## Setup NVIDIA Drivers

### Ubuntu 22.04

```bash
# Update system
apt update && apt upgrade -y

# Install NVIDIA driver
apt install -y nvidia-driver-535

# Reboot
reboot

# Verify
nvidia-smi
```

### Install CUDA Toolkit

```bash
# Add NVIDIA repo
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
dpkg -i cuda-keyring_1.1-1_all.deb
apt update

# Install CUDA
apt install -y cuda-toolkit-12-3

# Add to PATH
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc

# Verify
nvcc --version
```

### Install cuDNN

```bash
# Download from NVIDIA (requires account)
# Or install via apt
apt install -y libcudnn8 libcudnn8-dev
```

## Machine Learning Setup

### PyTorch

```bash
# Create virtual environment
python3 -m venv ml-env
source ml-env/bin/activate

# Install PyTorch with CUDA
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121

# Verify GPU
python -c "import torch; print(torch.cuda.is_available())"
python -c "import torch; print(torch.cuda.get_device_name(0))"
```

### TensorFlow

```bash
# Install TensorFlow GPU
pip install tensorflow[and-cuda]

# Verify GPU
python -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"
```

### JAX

```bash
pip install jax[cuda12_pip] -f https://storage.googleapis.com/jax-releases/jax_cuda_releases.html

python -c "import jax; print(jax.devices())"
```

## Example: Training a Model

### PyTorch Training Script

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader
from torchvision import datasets, transforms

# Check GPU
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
print(f'Using device: {device}')

# Simple model
class Net(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(784, 256)
        self.fc2 = nn.Linear(256, 10)

    def forward(self, x):
        x = x.view(-1, 784)
        x = torch.relu(self.fc1(x))
        return self.fc2(x)

# Data
transform = transforms.ToTensor()
train_data = datasets.MNIST('./data', train=True, download=True, transform=transform)
train_loader = DataLoader(train_data, batch_size=64, shuffle=True)

# Train
model = Net().to(device)
optimizer = optim.Adam(model.parameters())
criterion = nn.CrossEntropyLoss()

for epoch in range(5):
    for batch_idx, (data, target) in enumerate(train_loader):
        data, target = data.to(device), target.to(device)
        optimizer.zero_grad()
        output = model(data)
        loss = criterion(output, target)
        loss.backward()
        optimizer.step()
    print(f'Epoch {epoch+1}, Loss: {loss.item():.4f}')
```

### Multi-GPU Training

```python
import torch
import torch.nn as nn
from torch.nn.parallel import DataParallel

# Check available GPUs
n_gpus = torch.cuda.device_count()
print(f'Available GPUs: {n_gpus}')

# Wrap model with DataParallel
model = Net()
if n_gpus > 1:
    model = DataParallel(model)
model = model.to('cuda')

# Training uses all GPUs automatically
```

## Docker with GPU

```bash
# Install NVIDIA Container Toolkit
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list > /etc/apt/sources.list.d/nvidia-docker.list
apt update && apt install -y nvidia-container-toolkit
systemctl restart docker

# Run container with GPU
docker run --gpus all nvidia/cuda:12.3-runtime-ubuntu22.04 nvidia-smi

# Run PyTorch container
docker run --gpus all pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime python -c "import torch; print(torch.cuda.is_available())"
```

## Monitoring GPU

### nvidia-smi

```bash
# One-time status
nvidia-smi

# Continuous monitoring
watch -n 1 nvidia-smi

# GPU utilization
nvidia-smi --query-gpu=utilization.gpu --format=csv -l 1

# Memory usage
nvidia-smi --query-gpu=memory.used,memory.total --format=csv
```

### nvtop

```bash
apt install nvtop
nvtop
```

## Best Practices

```
1. Cost Optimization
   +-- Use Nanode for development, GPU for training
   +-- Snapshot trained models to Object Storage
   +-- Shutdown GPU instances when not training
   +-- Use checkpointing for long training runs

2. Performance
   +-- Use NVMe storage for datasets
   +-- Enable mixed precision training (FP16)
   +-- Optimize batch size for GPU memory
   +-- Use DataLoader with multiple workers

3. Data Management
   +-- Store datasets on Block Storage
   +-- Use Object Storage for model artifacts
   +-- Pre-process data on CPU instances

4. Reliability
   +-- Save checkpoints frequently
   +-- Use spot instances with checkpointing
   +-- Monitor GPU temperature and utilization
```

## Gotchas

- GPU instances are dedicated (no sharing)
- Limited region availability
- Requires manual driver installation (no pre-installed)
- vGPU not supported (full GPU only)
- Higher cost than cloud GPU alternatives
- Must reboot after driver installation
- Some ML frameworks need specific CUDA versions
- Container runtime needs NVIDIA toolkit

## Limits

| Resource | Limit |
|----------|-------|
| GPUs per instance | 1-2 (depending on plan) |
| GPU RAM | 8-48 GB |
| System RAM | 32-64 GB |
| vCPUs | 8-16 |
| Storage | 640 GB - 1.28 TB NVMe |
