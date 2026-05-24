# The Complete Guide to Modern AI Infrastructure: From Request to Response

> **The definitive technical deep-dive into how AI systems like ChatGPT, Claude, Gemini, and Grok actually work—from the silicon to the software.**

---

## The Standard AI Infrastructure Architecture

This is the reference architecture used by OpenAI, Anthropic, Google, Meta, and xAI for serving AI at scale:

```
+==================================================================================+
|                                                                                  |
|                            [1] GLOBAL EDGE LAYER                                 |
|                                                                                  |
|     +----------+    +----------+    +----------+    +----------+    +----------+ |
|     | CDN PoP  |    | CDN PoP  |    | CDN PoP  |    | CDN PoP  |    | CDN PoP  | |
|     | US-West  |    | US-East  |    | Europe   |    | Asia     |    | Aus/NZ   | |
|     +----+-----+    +----+-----+    +----+-----+    +----+-----+    +----+-----+ |
|          |              |              |              |              |           |
|          +--------------+--------------+--------------+--------------+           |
|                                    |                                             |
|              [TLS Termination | DDoS Protection | Rate Limiting]                 |
|                                    |                                             |
|                                    v                                             |
+==================================================================================+
|                                                                                  |
|                            [2] API GATEWAY LAYER                                 |
|                                                                                  |
|  +------------------------------------------------------------------------+      |
|  |                                                                        |      |
|  |  +-----------+    +-----------+    +-----------+    +-----------+      |      |
|  |  |   Auth    |--->|   Quota   |--->|  Request  |--->|  Billing  |      |      |
|  |  |  Service  |    |  Manager  |    | Validator |    |  Metering |      |      |
|  |  | (API Key) |    | (Limits)  |    | (Schema)  |    | (Tokens)  |      |      |
|  |  +-----------+    +-----------+    +-----------+    +-----------+      |      |
|  |                                                                        |      |
|  +------------------------------------------------------------------------+      |
|                                    |                                             |
|                                    v                                             |
+==================================================================================+
|                                                                                  |
|                         [3] INTELLIGENT ROUTING LAYER                            |
|                                                                                  |
|  +------------------------------------------------------------------------+      |
|  |                     MODEL ROUTER / SMART GATEWAY                       |      |
|  |                                                                        |      |
|  |   * Complexity Detection (simple query vs deep reasoning)              |      |
|  |   * Model Selection (GPT-4 vs GPT-4o, Claude Opus vs Sonnet)           |      |
|  |   * KV-Cache Aware Routing (route to server with cached context)       |      |
|  |   * Load-Based Distribution (GPU utilization, queue depth)             |      |
|  |   * Latency Optimization (TTFT targets)                                |      |
|  +------------------------------------------------------------------------+      |
|                                    |                                             |
|           +------------------------+------------------------+                    |
|           |                        |                        |                    |
|           v                        v                        v                    |
|    +------------+           +------------+           +------------+              |
|    | Fast Model |           |  Standard  |           | Reasoning  |              |
|    | (Low TTFT) |           |   Model    |           |   Model    |              |
|    +------------+           +------------+           +------------+              |
|                                    |                                             |
|                                    v                                             |
+==================================================================================+
|                                                                                  |
|                         [4] INFERENCE CLUSTER LAYER                              |
|                                                                                  |
|  +------------------------------------------------------------------------+      |
|  |                        KUBERNETES CLUSTER                              |      |
|  |                                                                        |      |
|  |  +------------------+  +------------------+  +------------------+      |      |
|  |  |  Inference Pod   |  |  Inference Pod   |  |  Inference Pod   |      |      |
|  |  |                  |  |                  |  |                  |      |      |
|  |  |  +------------+  |  |  +------------+  |  |  +------------+  |      |      |
|  |  |  | vLLM /     |  |  |  | vLLM /     |  |  |  | vLLM /     |  |      |      |
|  |  |  | TensorRT   |  |  |  | TensorRT   |  |  |  | TensorRT   |  |      |      |
|  |  |  +------------+  |  |  +------------+  |  |  +------------+  |      |      |
|  |  |  +------------+  |  |  +------------+  |  |  +------------+  |      |      |
|  |  |  | KV Cache   |  |  |  | KV Cache   |  |  |  | KV Cache   |  |      |      |
|  |  |  +------------+  |  |  +------------+  |  |  +------------+  |      |      |
|  |  +------------------+  +------------------+  +------------------+      |      |
|  |                                                                        |      |
|  |  [HPA: GPU Util] [VPA: Memory] [Cluster Autoscaler] [Prometheus]       |      |
|  +------------------------------------------------------------------------+      |
|                                    |                                             |
|                                    v                                             |
+==================================================================================+
|                                                                                  |
|                            [5] GPU SERVER LAYER                                  |
|                                                                                  |
|  +------------------------------------------------------------------------+      |
|  |                        8-GPU SERVER (DGX/HGX)                          |      |
|  |                                                                        |      |
|  |  +------+  +------+  +------+  +------+  +------+  +------+  +------+  |      |
|  |  | H100 |  | H100 |  | H100 |  | H100 |  | H100 |  | H100 |  | H100 |  |      |
|  |  | 80GB |  | 80GB |  | 80GB |  | 80GB |  | 80GB |  | 80GB |  | 80GB |  |      |
|  |  +--+---+  +--+---+  +--+---+  +--+---+  +--+---+  +--+---+  +--+---+  |      |
|  |     |         |         |         |         |         |         |      |      |
|  |     +----+----+----+----+----+----+----+----+----+----+----+----+      |      |
|  |                              |                                         |      |
|  |                     +--------+--------+                                |      |
|  |                     | NVLink/NVSwitch |                                |      |
|  |                     |   (1.8 TB/s)    |                                |      |
|  |                     +-----------------+                                |      |
|  |                                                                        |      |
|  |  [Memory: 640GB HBM3] [Compute: 15,832 TFLOPS FP8] [Power: 5.6 kW]     |      |
|  +------------------------------------------------------------------------+      |
|                                    |                                             |
|                                    v                                             |
+==================================================================================+
|                                                                                  |
|                        [6] HIGH-SPEED NETWORK FABRIC                             |
|                                                                                  |
|  +------------------------------------------------------------------------+      |
|  |                             SPINE LAYER                                |      |
|  |        +----------------------------------------------------+          |      |
|  |        | 800 Gbps Switches (InfiniBand / Spectrum-X / RoCE) |          |      |
|  |        +-----+--------------------+--------------------+----+          |      |
|  |              |                    |                    |               |      |
|  |              v                    v                    v               |      |
|  |        +----------------------------------------------------+          |      |
|  |        |   LEAF LAYER: 400 Gbps ToR Switches (Top of Rack)  |          |      |
|  |        +-------+------------------+------------------+------+          |      |
|  |                |                  |                  |                 |      |
|  |                v                  v                  v                 |      |
|  |         +-----------+      +-----------+      +-----------+            |      |
|  |         |  Rack 1   |      |  Rack 2   |      |  Rack N   |            |      |
|  |         | 8x Nodes  |      | 8x Nodes  |      | 8x Nodes  |            |      |
|  |         | (64 GPUs) |      | (64 GPUs) |      | (64 GPUs) |            |      |
|  |         +-----------+      +-----------+      +-----------+            |      |
|  |                                                                        |      |
|  |  [Latency: 1-2 us] [Throughput: 95%+ efficiency] [Zero packet loss]    |      |
|  +------------------------------------------------------------------------+      |
|                                    |                                             |
|                                    v                                             |
+==================================================================================+
|                                                                                  |
|                        [7] DATA CENTER INFRASTRUCTURE                            |
|                                                                                  |
|  +--------------------+  +--------------------+  +--------------------+          |
|  |       POWER        |  |      COOLING       |  |      SECURITY      |          |
|  +--------------------+  +--------------------+  +--------------------+          |
|  | * 100-500 MW       |  | * Direct Liquid    |  | * Physical access  |          |
|  | * Redundant feeds  |  |   Cooling (DLC)    |  | * Network isolation|          |
|  | * UPS + Generators |  | * Cold plate GPUs  |  | * Encryption       |          |
|  | * Battery backup   |  | * PUE: 1.1-1.3     |  | * SOC monitoring   |          |
|  +--------------------+  +--------------------+  +--------------------+          |
|                                                                                  |
+==================================================================================+


+===================================================================================+
|                                                                                   |
|                              SUPPORTING SYSTEMS                                   |
|                                                                                   |
|  +------------------+ +------------------+ +------------------+ +---------------+ |
|  |  OBSERVABILITY   | |  MODEL STORAGE   | |  TRAINING INFRA  | | SAFETY SYSTEMS| |
|  +------------------+ +------------------+ +------------------+ +---------------+ |
|  | * Prometheus     | | * Model Registry | | * GPU Clusters   | | * Filters     | |
|  | * Grafana        | | * S3/GCS/Azure   | | * Checkpointing  | | * Rate Limits | |
|  | * GPU metrics    | | * Version Control| | * Lustre/GPFS    | | * Red Teaming | |
|  | * Request traces | | * A/B Deployment | | * Data Pipeline  | | * Validation  | |
|  +------------------+ +------------------+ +------------------+ +---------------+ |
|                                                                                   |
+===================================================================================+
```

### Request Flow Summary

```
+--------+     +---------+     +-------------+     +--------------+     +------------+
|  User  |---->| CDN/    |---->| API Gateway |---->| Smart Router |---->| Inference  |
|        |     | Edge    |     |             |     |              |     | Server     |
+--------+     +---------+     +-------------+     +--------------+     +-----+------+
                   |                |                    |                    |
               TLS/DDoS        Auth/Quota          Model Select         Tokenize
                                                                        KV Cache
                                                                             |
                                                                             v
+--------+     +---------+     +-------------+     +--------------+     +------------+
|  User  |<----| Stream  |<----| API Gateway |<----| Smart Router |<----| GPU        |
|  Sees  |     | Tokens  |     |             |     |              |     | Cluster    |
| Answer |     |         |     |             |     |              |     | (Attention)|
+--------+     +---------+     +-------------+     +--------------+     +------------+
```

---

## Table of Contents

1. [Introduction: What is AI?](#1-introduction-what-is-ai)
2. [Understanding Neural Networks](#2-understanding-neural-networks)
3. [The Transformer Architecture](#3-the-transformer-architecture)
4. [How Models Are Trained](#4-how-models-are-trained)
5. [The Hardware: GPUs, TPUs, and Custom Silicon](#5-the-hardware-gpus-tpus-and-custom-silicon)
6. [Data Center Infrastructure](#6-data-center-infrastructure)
7. [The Request Journey: What Happens When You Ask AI a Question](#7-the-request-journey)
8. [Model Serving and Inference](#8-model-serving-and-inference)
9. [Company Deep-Dives](#9-company-deep-dives)
10. [Safety, Alignment, and Testing](#10-safety-alignment-and-testing)
11. [The Future of AI Infrastructure](#11-the-future-of-ai-infrastructure)

---

## 1. Introduction: What is AI?

### 1.1 Artificial Intelligence Defined

Artificial Intelligence (AI) refers to computer systems designed to perform tasks that typically require human intelligence. Modern AI, specifically **Large Language Models (LLMs)**, are statistical models that predict the next word (or token) in a sequence based on patterns learned from vast amounts of text data.

### 1.2 Key Terminology

| Term | Definition |
|------|------------|
| **Model** | A mathematical function with billions of parameters that transforms input into output |
| **Parameters** | The learnable weights in a neural network (GPT-4 has ~1.8 trillion parameters) |
| **Training** | The process of adjusting parameters using data to improve predictions |
| **Inference** | Using a trained model to generate predictions/responses |
| **Token** | The basic unit of text processing (roughly 4 characters or 0.75 words) |
| **Context Window** | Maximum tokens a model can process at once (ranging from 8K to 1M+) |

### 1.3 The AI Stack Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER APPLICATION                         │
│                   (ChatGPT, Claude, Gemini UI)                  │
├─────────────────────────────────────────────────────────────────┤
│                          API LAYER                              │
│              (REST APIs, WebSockets, Streaming)                 │
├─────────────────────────────────────────────────────────────────┤
│                      INFERENCE ENGINE                           │
│            (vLLM, TensorRT-LLM, Triton Server)                  │
├─────────────────────────────────────────────────────────────────┤
│                      MODEL WEIGHTS                              │
│         (Transformer parameters stored in GPU memory)           │
├─────────────────────────────────────────────────────────────────┤
│                    COMPUTE HARDWARE                             │
│              (NVIDIA H100/B200, Google TPUs)                    │
├─────────────────────────────────────────────────────────────────┤
│                  NETWORKING FABRIC                              │
│           (InfiniBand, NVLink, RoCE, Spectrum-X)                │
├─────────────────────────────────────────────────────────────────┤
│                    DATA CENTER                                  │
│        (Power, Cooling, Physical Infrastructure)                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Understanding Neural Networks

### 2.1 What is a Neural Network?

A neural network is a computational model inspired by the human brain. It consists of layers of interconnected nodes (neurons) that process information.

```
INPUT LAYER          HIDDEN LAYERS           OUTPUT LAYER
    ○                    ○                       ○
    ○ ──────────────► ○   ○ ──────────────►     ○
    ○                    ○                       ○
    ○ ──────────────► ○   ○ ──────────────►     ○
    ○                    ○
```

### 2.2 Key Components

**Neurons (Nodes)**
- Receive inputs, apply weights, sum them, and pass through an activation function
- Formula: `output = activation(Σ(input × weight) + bias)`

**Weights and Biases**
- **Weights**: Multipliers that determine the importance of each input
- **Biases**: Constants added to shift the activation function

**Activation Functions**
- Introduce non-linearity to learn complex patterns
- Common types: ReLU, GELU, Sigmoid, Softmax

### 2.3 How Training Works: Backpropagation and Gradient Descent

Training is the process of finding optimal parameter values:

```
┌─────────────────────────────────────────────────────────────┐
│                    TRAINING LOOP                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. FORWARD PASS                                            │
│     Input ──► Network ──► Prediction                        │
│                                                             │
│  2. LOSS CALCULATION                                        │
│     Loss = difference(Prediction, Actual)                   │
│                                                             │
│  3. BACKWARD PASS (Backpropagation)                         │
│     Calculate gradient of loss w.r.t. each weight           │
│     Using chain rule: ∂L/∂w = ∂L/∂y × ∂y/∂w                 │
│                                                             │
│  4. WEIGHT UPDATE (Gradient Descent)                        │
│     new_weight = old_weight - (learning_rate × gradient)    │
│                                                             │
│  5. REPEAT for millions/billions of examples                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Key Insight**: Backpropagation computes *which direction* to adjust each parameter, and gradient descent *applies* that adjustment. Together, they're the engine behind all modern deep learning.

---

## 3. The Transformer Architecture

### 3.1 The Revolution: "Attention Is All You Need"

The Transformer architecture, introduced by Vaswani et al. in 2017, revolutionized AI by enabling parallel processing of sequences and capturing long-range dependencies.

### 3.2 Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                  TRANSFORMER BLOCK (×N)                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              MULTI-HEAD ATTENTION                   │    │
│  │                                                     │    │
│  │   Q (Query)  ───┐                                   │    │
│  │   K (Key)    ───┼──► Attention Scores ──► Output    │    │
│  │   V (Value)  ───┘                                   │    │
│  │                                                     │    │
│  │   Attention(Q,K,V) = softmax(QK^T/√d) × V           │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                  │
│                    Add & Normalize                          │
│                          │                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              FEED-FORWARD NETWORK                   │    │
│  │         Linear ──► GELU ──► Linear                  │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                  │
│                    Add & Normalize                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 The Attention Mechanism Explained

Self-attention allows each token to "look at" all other tokens in the sequence:

1. **Query (Q)**: "What am I looking for?"
2. **Key (K)**: "What do I contain?"
3. **Value (V)**: "What information do I provide?"

```
Example: "The cat sat on the mat"

For the word "sat":
- High attention to "cat" (subject)
- High attention to "mat" (location)
- Lower attention to "the" (less relevant)
```

### 3.4 Multi-Head Attention

Instead of one attention mechanism, transformers use multiple "heads" (typically 32-128) that learn different types of relationships:
- Head 1: Subject-verb relationships
- Head 2: Positional patterns
- Head 3: Semantic similarity
- etc.

### 3.5 Tokenization: From Text to Numbers

Before processing, text must be converted to tokens:

```
┌─────────────────────────────────────────────────────────────┐
│                   TOKENIZATION PIPELINE                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. RAW TEXT                                                │
│     "Hello, how are you?"                                   │
│              │                                              │
│              ▼                                              │
│  2. TOKENIZE (BPE/WordPiece)                                │
│     ["Hello", ",", "how", "are", "you", "?"]                │
│              │                                              │
│              ▼                                              │
│  3. TOKEN IDs                                               │
│     [15496, 11, 9257, 389, 345, 30]                         │
│              │                                              │
│              ▼                                              │
│  4. EMBEDDINGS (learned vectors)                            │
│     [[0.23, -0.45, ...], [0.12, 0.89, ...], ...]            │
│     Each token → 4096-12288 dimensional vector              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Token Economics**:
- ~100K vocabulary size for modern models
- 1 token ≈ 4 characters ≈ 0.75 words
- Cost is typically per-token (input + output)

---

## 4. How Models Are Trained

### 4.1 The Three Stages of LLM Training

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  STAGE 1: PRE-TRAINING                                      │
│  ════════════════════                                       │
│  • Dataset: Trillions of tokens from internet, books, code  │
│  • Objective: Predict next token                            │
│  • Duration: Weeks to months                                │
│  • Compute: 10,000-100,000+ GPUs                            │
│  • Cost: $10M - $500M+                                      │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  STAGE 2: SUPERVISED FINE-TUNING (SFT)                      │
│  ══════════════════════════════════════                     │
│  • Dataset: High-quality instruction-response pairs         │
│  • Objective: Learn to follow instructions                  │
│  • Human contractors write example conversations            │
│  • Duration: Days to weeks                                  │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  STAGE 3: RLHF (Reinforcement Learning from Human Feedback) │
│  ═══════════════════════════════════════════════════════════│
│  • Train reward model from human preferences                │
│  • Use PPO/DPO to optimize for preferred responses          │
│  • Balance helpfulness vs harmlessness                      │
│  • Duration: Weeks                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Distributed Training: Parallelism Strategies

Training modern LLMs requires distributing work across thousands of GPUs:

```
┌─────────────────────────────────────────────────────────────┐
│              DISTRIBUTED TRAINING STRATEGIES                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  DATA PARALLELISM                                           │
│  ─────────────────                                          │
│  • Same model copied to each GPU                            │
│  • Different data batches per GPU                           │
│  • Gradients synchronized across GPUs                       │
│  • Most common approach                                     │
│                                                             │
│  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐                         │
│  │GPU 0│  │GPU 1│  │GPU 2│  │GPU 3│                         │
│  │Model│  │Model│  │Model│  │Model│  ◄── Same model         │
│  │Data1│  │Data2│  │Data3│  │Data4│  ◄── Different data     │
│  └─────┘  └─────┘  └─────┘  └─────┘                         │
│      │       │        │        │                            │
│      └───────┴────────┴────────┘                            │
│              │                                              │
│         Sync Gradients                                      │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  MODEL PARALLELISM (Tensor Parallelism)                     │
│  ──────────────────────────────────────                     │
│  • Model split across GPUs                                  │
│  • Each GPU holds part of each layer                        │
│  • Required when model > single GPU memory                  │
│                                                             │
│  ┌─────────────────────────────────────┐                    │
│  │            LAYER N                  │                    │
│  │  ┌────┐ ┌────┐ ┌────┐ ┌────┐        │                    │
│  │  │GPU0│ │GPU1│ │GPU2│ │GPU3│        │                    │
│  │  │25% │ │25% │ │25% │ │25% │        │                    │
│  │  └────┘ └────┘ └────┘ └────┘        │                    │
│  └─────────────────────────────────────┘                    │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  PIPELINE PARALLELISM                                       │
│  ────────────────────                                       │
│  • Different layers on different GPUs                       │
│  • Data flows through pipeline                              │
│  • Micro-batching to reduce bubble overhead                 │
│                                                             │
│  GPU 0: [Layer 1-10]  ──►                                   │
│  GPU 1: [Layer 11-20] ──►                                   │
│  GPU 2: [Layer 21-30] ──►                                   │
│  GPU 3: [Layer 31-40] ──►                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 Real-World Training Scale

| Model | Parameters | Training Tokens | GPUs | Training Time |
|-------|-----------|-----------------|------|---------------|
| GPT-3 | 175B | 300B | ~10,000 V100s | ~34 days |
| Llama 2 70B | 70B | 2T | 2,048 A100s | ~21 days |
| Llama 3 405B | 405B | 15T | 16,000 H100s | ~54 days |
| GPT-4 | ~1.8T (MoE) | 13T+ | 25,000+ A100s | ~100 days |

**Example**: Stable Diffusion training took 150,000 GPU hours (17 years on 1 GPU), but parallel training across 256 GPUs reduced it to just 25 days.

---

## 5. The Hardware: GPUs, TPUs, and Custom Silicon

### 5.1 NVIDIA GPU Evolution

```
┌─────────────────────────────────────────────────────────────┐
│                  NVIDIA DATA CENTER GPUs                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  A100 (2020) - Ampere Architecture                          │
│  ─────────────────────────────────                          │
│  • Memory: 40GB / 80GB HBM2e                                │
│  • Bandwidth: 2 TB/s                                        │
│  • FP16: 312 TFLOPS                                         │
│  • Use: Baseline for most training                          │
│                                                             │
│  H100 (2022) - Hopper Architecture                          │
│  ─────────────────────────────────                          │
│  • Memory: 80GB HBM3                                        │
│  • Bandwidth: 3.35 TB/s                                     │
│  • FP16: 1,979 TFLOPS                                       │
│  • FP8: 3,958 TFLOPS (Transformer Engine)                   │
│  • Use: Current production standard                         │
│                                                             │
│  H200 (2024) - Hopper Architecture (Memory Enhanced)        │
│  ───────────────────────────────────────────────            │
│  • Memory: 141GB HBM3e (+76% vs H100)                       │
│  • Bandwidth: 4.8 TB/s (+43% vs H100)                       │
│  • 2x inference speed for LLMs vs H100                      │
│  • Use: Large context inference, KV cache heavy workloads   │
│                                                             │
│  B200 (2025) - Blackwell Architecture                       │
│  ────────────────────────────────────                       │
│  • Memory: 192GB HBM3e                                      │
│  • Bandwidth: 8 TB/s (2x H100)                              │
│  • 208 billion transistors (dual-die)                       │
│  • Training: 3x faster than H100                            │
│  • Inference: 15x faster than H100                          │
│  • Native FP4 support                                       │
│  • Use: Next-gen training and inference                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 GPU Specifications Comparison

| Spec | A100 80GB | H100 SXM | H200 | B200 |
|------|-----------|----------|------|------|
| Architecture | Ampere | Hopper | Hopper | Blackwell |
| Memory | 80GB HBM2e | 80GB HBM3 | 141GB HBM3e | 192GB HBM3e |
| Memory BW | 2.0 TB/s | 3.35 TB/s | 4.8 TB/s | 8.0 TB/s |
| FP16 TFLOPS | 312 | 1,979 | 1,979 | 2,250+ |
| FP8 TFLOPS | N/A | 3,958 | 3,958 | 4,500+ |
| TDP | 400W | 700W | 700W | 1000W |
| Price (est.) | $10K | $30K | $40K | $50K+ |

### 5.3 Google TPUs (Tensor Processing Units)

```
┌─────────────────────────────────────────────────────────────┐
│                    GOOGLE TPU GENERATIONS                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  TPU v4 (2021)                                              │
│  • 275 TFLOPS (BF16)                                        │
│  • 32GB HBM2e                                               │
│  • Used for: PaLM, original Gemini                          │
│                                                             │
│  TPU v5p (2023)                                             │
│  • 459 TFLOPS (BF16)                                        │
│  • 95GB HBM2e                                               │
│  • 2x training speed vs v4                                  │
│  • Used for: Gemini 1.5                                     │
│                                                             │
│  Trillium / TPU v6e (2024)                                  │
│  • 4.7x compute vs v5e                                      │
│  • 2x memory and interconnect bandwidth                     │
│  • Used for: Gemini 2.0                                     │
│                                                             │
│  Ironwood / TPU v7 (2025)                                   │
│  • 4,614 TFLOPS peak                                        │
│  • Configurations: 256-chip and 9,216-chip clusters         │
│  • Used for: Gemini 3.0 and beyond                          │
│                                                             │
│  TPU v8 (2025+)                                             │
│  • First bifurcated design: TPU 8t (training), TPU 8i       │
│  • TPU 8i: Optimized for inference with expanded SRAM       │
│  • On-silicon KV cache hosting                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.4 AWS Custom Silicon: Trainium

| Chip | Purpose | Performance | Cost Advantage |
|------|---------|-------------|----------------|
| Trainium1 | Training | Comparable to A100 | 50% cheaper |
| Trainium2 | Training | Comparable to H100 | ~50% cheaper |
| Inferentia2 | Inference | Optimized throughput | Up to 4x cost effective |

Anthropic runs one of the world's largest Trainium2 clusters for training Claude.

---

## 6. Data Center Infrastructure

### 6.1 Networking: The Backbone of AI Clusters

```
┌─────────────────────────────────────────────────────────────┐
│              AI DATA CENTER NETWORK HIERARCHY               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  INTRA-NODE (Within a Server)                               │
│  ════════════════════════════                               │
│  NVLink: 1.8 TB/s bidirectional per GPU                     │
│  • Connects up to 8 GPUs within single server               │
│  • 14x bandwidth of PCIe Gen 5                              │
│  • Fifth generation supports 72 GPUs per rack               │
│                                                             │
│     ┌─────┐  NVLink   ┌─────┐                               │
│     │GPU 0│◄─────────►│GPU 1│                               │
│     └─────┘  1.8TB/s  └─────┘                               │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  INTER-NODE (Between Servers)                               │
│  ════════════════════════════                               │
│                                                             │
│  Option 1: InfiniBand                                       │
│  • 400-800 Gb/s per port                                    │
│  • 1-2 microsecond latency                                  │
│  • Best for: Tightly-coupled training                       │
│  • Used by: xAI Colossus, Meta (partial)                    │
│                                                             │
│  Option 2: RoCE (RDMA over Converged Ethernet)              │
│  • 400-800 Gb/s                                             │
│  • Slightly higher latency than IB                          │
│  • More flexible, lower cost                                │
│  • Used by: Meta, Microsoft, AWS, Google                    │
│                                                             │
│  Option 3: Spectrum-X (NVIDIA)                              │
│  • 800 Gb/s Ethernet                                        │
│  • 95% data throughput, zero packet loss                    │
│  • AI-optimized congestion control                          │
│  • Used by: xAI Colossus                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 Network Topology

```
                           SPINE LAYER
            ┌───────────────────────────────────────┐
            │   High-bandwidth switches (800G)      │
            └─────────┬─────────┬─────────┬─────────┘
                      │         │         │
            ┌─────────┴─────────┴─────────┴─────────┐
            │              LEAF LAYER               │
            └─────┬─────┬─────┬─────┬─────┬─────────┘
                  │     │     │     │     │     
              ┌───┴───┐ │ ┌───┴───┐ │ ┌───┴───┐ 
              │ Rack 1│ │ │ Rack 2│ │ │ Rack N│ 
              │8x H100│ │ │8x H100│ │ │8x H100│ 
              └───────┘   └───────┘   └───────┘
```

### 6.3 Power and Cooling Requirements

| Scale | GPUs | Power | Cooling | Cost/Year |
|-------|------|-------|---------|-----------|
| Small Cluster | 64 H100s | 100 kW | Air cooling | $500K |
| Medium Cluster | 1,000 H100s | 1 MW | Liquid cooling | $10M |
| Large Cluster | 10,000 H100s | 10 MW | Liquid cooling | $100M |
| Hyperscale | 100,000+ H100s | 100+ MW | Custom datacenter | $1B+ |

**xAI Colossus**: 250-300 MW power consumption with 168 Tesla Megapacks for backup.

### 6.4 Major AI Infrastructure Deployments

| Company | Facility | GPUs | Network | Power |
|---------|----------|------|---------|-------|
| xAI | Colossus (Memphis) | 200,000+ H100/H200 | Spectrum-X 800G | 250-300 MW |
| Meta | 2x Clusters | 2×24,576 H100s | RoCE + InfiniBand | ~100 MW |
| OpenAI | Stargate (Texas) | GB200 systems | Oracle Cloud | Multi-GW |
| Anthropic | AWS + SpaceX | 220,000 H100s (leased) | AWS/Trainium2 | ~200 MW |
| Google | TPU Pods | Millions of TPU chips | Custom fabric | Multi-GW |

---

## 7. The Request Journey: What Happens When You Ask AI a Question

### 7.1 End-to-End Request Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE REQUEST LIFECYCLE                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  USER                                                                   │
│    │                                                                    │
│    │ 1. "Explain quantum computing"                                     │
│    ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                        EDGE/CDN LAYER                            │   │
│  │  • TLS termination                                               │   │
│  │  • DDoS protection                                               │   │
│  │  • Geographic routing                                            │   │
│  │  • Rate limiting                                                 │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│    │                                                                    │
│    │ 2. HTTPS request to api.openai.com / api.anthropic.com             │
│    ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                       API GATEWAY                                │   │
│  │  • Authentication (API key validation)                           │   │
│  │  • Authorization (quota, tier limits)                            │   │
│  │  • Request validation                                            │   │
│  │  • Billing/metering                                              │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│    │                                                                    │
│    │ 3. Authenticated request with metadata                             │
│    ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     LOAD BALANCER                                │   │
│  │  • Health checking                                               │   │
│  │  • Session affinity (for streaming)                              │   │
│  │  • Model-aware routing (GPT-4 vs GPT-3.5)                        │   │
│  │  • KV-cache aware routing (advanced)                             │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│    │                                                                    │
│    │ 4. Route to optimal inference server                               │
│    ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                   INFERENCE SERVER                               │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │                    TOKENIZER                               │  │   │
│  │  │  "Explain quantum computing" → [50268, 12345, 9876, ...]   │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  │                          │                                       │   │
│  │                          ▼                                       │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │                  EMBEDDING LOOKUP                          │  │   │
│  │  │  Token IDs → Dense vectors (4096-12288 dimensions)         │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  │                          │                                       │   │
│  │                          ▼                                       │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │                GPU INFERENCE ENGINE                        │  │   │
│  │  │  ┌──────────────────────────────────────────────────────┐  │  │   │
│  │  │  │  PREFILL PHASE (Process input)                       │  │  │   │
│  │  │  │  • All input tokens processed in parallel            │  │  │   │
│  │  │  │  • KV cache populated                                │  │  │   │
│  │  │  │  • GPU-bound (compute intensive)                     │  │  │   │
│  │  │  └──────────────────────────────────────────────────────┘  │  │   │
│  │  │                          │                                 │  │   │
│  │  │                          ▼                                 │  │   │
│  │  │  ┌──────────────────────────────────────────────────────┐  │  │   │
│  │  │  │  DECODE PHASE (Generate output)                      │  │  │   │
│  │  │  │  • One token generated at a time                     │  │  │   │
│  │  │  │  • Each token attends to KV cache                    │  │  │   │
│  │  │  │  • Memory-bound (bandwidth intensive)                │  │  │   │
│  │  │  │  • Repeat until EOS or max_tokens                    │  │  │   │
│  │  │  └──────────────────────────────────────────────────────┘  │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  │                          │                                       │   │
│  │                          ▼                                       │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │                    DETOKENIZER                             │  │   │
│  │  │  [98765, 54321, ...] → "Quantum computing is..."           │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│    │                                                                    │
│    │ 5. Stream tokens back (Server-Sent Events)                         │
│    ▼                                                                    │
│  USER sees: "Quantum computing is a type of computation that..."        │
│             (appearing word by word)                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Latency Breakdown

| Stage | Typical Latency | Notes |
|-------|-----------------|-------|
| Network round-trip | 20-100ms | Depends on geography |
| API Gateway | 5-20ms | Auth, validation |
| Queue wait | 0-500ms | Depends on load |
| Tokenization | 1-5ms | CPU operation |
| Prefill (TTFT) | 50-500ms | Time to first token |
| Per-token decode | 10-50ms | ~20-100 tokens/sec |
| **Total for 500 tokens** | **2-10 seconds** | End-to-end |

### 7.3 The KV Cache: Why Memory Matters

```
┌─────────────────────────────────────────────────────────────┐
│                      KV CACHE EXPLAINED                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Without KV Cache:                                          │
│  ─────────────────                                          │
│  Token 1: Process [Token 1]                                 │
│  Token 2: Process [Token 1, Token 2]          ◄── Redundant │
│  Token 3: Process [Token 1, Token 2, Token 3] ◄── Redundant │
│  ...                                                        │
│  Token N: Process all N tokens                ◄── O(N²)!    │
│                                                             │
│  With KV Cache:                                             │
│  ──────────────                                             │
│  Token 1: Compute K,V → Store in cache                      │
│  Token 2: Compute K,V → Store, attend to cached K,V         │
│  Token 3: Compute K,V → Store, attend to cached K,V         │
│  ...                                                        │
│  Token N: Only compute new token, reuse cache  ◄── O(N)     │
│                                                             │
│  Memory per request: 2 × layers × heads × dim × seq_len     │
│  Example (70B model, 4K context):                           │
│    2 × 80 × 64 × 128 × 4096 × 2 bytes = 5.4 GB per request  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Why H200/B200 matter**: More HBM = more KV cache = more concurrent requests = higher throughput.

---

## 8. Model Serving and Inference

### 8.1 Inference Frameworks Comparison

| Framework | Best For | Key Features | Throughput |
|-----------|----------|--------------|------------|
| **vLLM** | General production | PagedAttention, continuous batching, OpenAI-compatible | High |
| **TensorRT-LLM** | NVIDIA hardware | FP8 optimization, deep CUDA integration | Highest |
| **Triton Server** | Enterprise | Model ensemble, A/B testing, multi-framework | High |
| **TGI** | HuggingFace models | Easy deployment, watermarking, token streaming | Medium-High |
| **llama.cpp** | Edge/local | CPU inference, quantization, minimal dependencies | Medium |

### 8.2 Key Optimization Techniques

```
┌─────────────────────────────────────────────────────────────┐
│              INFERENCE OPTIMIZATION STACK                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. CONTINUOUS BATCHING                                     │
│     • Don't wait for batch to fill                          │
│     • Add new requests as slots free up                     │
│     • 2-10x throughput vs static batching                   │
│                                                             │
│  2. PAGED ATTENTION (vLLM)                                  │
│     • KV cache in non-contiguous memory blocks              │
│     • Eliminates memory fragmentation                       │
│     • Near-optimal memory utilization                       │
│                                                             │
│  3. QUANTIZATION                                            │
│     ┌────────┬──────────┬───────────┬──────────────┐        │
│     │ Format │ Mem Save │ Speed Up  │ Quality Loss │        │
│     ├────────┼──────────┼───────────┼──────────────┤        │
│     │ FP16   │ 2x       │ 2x        │ None         │        │
│     │ FP8    │ 4x       │ 2-3x      │ <1%          │        │
│     │ INT8   │ 4x       │ 2-3x      │ 1-3%         │        │
│     │ INT4   │ 8x       │ 2-4x      │ 3-5%         │        │
│     └────────┴──────────┴───────────┴──────────────┘        │
│                                                             │
│  4. SPECULATIVE DECODING                                    │
│     • Small "draft" model generates candidates              │
│     • Large model verifies in parallel                      │
│     • 2-3x speedup for some workloads                       │
│                                                             │
│  5. KV CACHE COMPRESSION                                    │
│     • FP8 KV cache: 50% memory savings                      │
│     • Enables 2x concurrent requests                        │
│                                                             │
│  6. FLASH ATTENTION                                         │
│     • Fused attention kernels                               │
│     • Reduced memory I/O                                    │
│     • 2-4x faster attention computation                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 8.3 Kubernetes and Autoscaling for AI

```yaml
# Example: Kubernetes HPA for LLM Inference
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: llm-inference-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: llm-inference
  minReplicas: 2
  maxReplicas: 100
  metrics:
  - type: Pods
    pods:
      metric:
        name: gpu_utilization
      target:
        type: AverageValue
        averageValue: "80"
  - type: Pods
    pods:
      metric:
        name: kv_cache_utilization
      target:
        type: AverageValue
        averageValue: "70"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
```

**Challenges unique to LLM autoscaling**:
- GPU cold start: 30-60 seconds to load model weights
- KV cache state: Can't easily migrate mid-request
- Memory-bound: Standard CPU metrics don't apply
- Solution: AI-aware routing (GKE Inference Gateway)

---

## 9. Company Deep-Dives

### 9.1 OpenAI

```
┌─────────────────────────────────────────────────────────────┐
│                       OPENAI INFRASTRUCTURE                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CLOUD PARTNERSHIP                                          │
│  • Primary: Microsoft Azure (exclusive)                     │
│  • Secondary: Oracle Cloud (Stargate project)               │
│                                                             │
│  HARDWARE EVOLUTION                                         │
│  • 2020-2022: Tens of thousands of V100s, A100s             │
│  • 2024: First batch of H200 GPUs                           │
│  • 2025: GB200 systems at Stargate (Abilene, Texas)         │
│                                                             │
│  ARCHITECTURAL INNOVATIONS                                  │
│  • GPT-4: Mixture of Experts (MoE) ~1.8T parameters         │
│  • GPT-5: Smart router system                               │
│    - Fast model for simple queries                          │
│    - Deep reasoning model for complex problems              │
│    - Real-time router based on complexity/intent            │
│                                                             │
│  CONTAINER ORCHESTRATION                                    │
│  • Kubernetes for model deployment                          │
│  • Auto-scaling based on traffic                            │
│                                                             │
│  STARGATE PROJECT (2025+)                                   │
│  • $500B investment over 4 years                            │
│  • Massive datacenter buildout                              │
│  • Oracle Cloud Infrastructure                              │
│  • Target: AGI-scale compute                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 9.2 Anthropic

```
┌─────────────────────────────────────────────────────────────┐
│                    ANTHROPIC INFRASTRUCTURE                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  MULTI-CLOUD STRATEGY                                       │
│  • Amazon Web Services (primary, $4B investment)            │
│  • Google Cloud (strategic investment)                      │
│  • Fluidstack (custom datacenter partner)                   │
│                                                             │
│  COMPUTE RESOURCES                                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  AWS Trainium2 Cluster                                │  │
│  │  • One of world's largest Trainium2 deployments       │  │
│  │  • ~50% cost of equivalent H100 capacity              │  │
│  │  • Primary Claude training workload                   │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  SpaceX Colossus 1 Lease (2025)                       │  │
│  │  • 220,000 NVIDIA H100/H200 GPUs                      │  │
│  │  • Location: Memphis, Tennessee                       │  │
│  │  • Upgrading to Blackwell GB200                       │  │
│  │  • Purpose: Address rate limit complaints             │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  $50 BILLION INFRASTRUCTURE INITIATIVE (Nov 2025)           │
│  • Custom datacenters in Texas and New York                 │
│  • Partnership with Fluidstack                              │
│  • First sites live in 2026                                 │
│  • Multi-gigawatt scale capacity                            │
│                                                             │
│  CLAUDE MODEL CHARACTERISTICS                               │
│  • Constitutional AI training approach                      │
│  • Focus on safety and helpfulness balance                  │
│  • Extensive red-teaming (153-page system card)             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 9.3 Google DeepMind (Gemini)

```
┌─────────────────────────────────────────────────────────────┐
│                  GOOGLE GEMINI INFRASTRUCTURE               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CUSTOM SILICON: TPU ADVANTAGE                              │
│  • Vertically integrated hardware + software                │
│  • Purpose-built for transformer workloads                  │
│  • Massive scale: Millions of TPU chips                     │
│                                                             │
│  AI HYPERCOMPUTER ARCHITECTURE (2023+)                      │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Hardware: TPU v5p, Trillium, Ironwood                │  │
│  │  Network: Jupiter fabric with optical switches        │  │
│  │  Software: XLA compiler, JAX, PyTorch/XLA             │  │
│  │  Orchestration: Custom pod management                 │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  GEMINI MODEL VARIANTS                                      │
│  • Gemini Ultra: Largest, most capable                      │
│  • Gemini Pro: Balanced performance                         │
│  • Gemini Flash: Speed + cost optimized                     │
│    - Latency comparable to simple search queries            │
│  • Gemini Nano: On-device (Android)                         │
│                                                             │
│  SCALE BENCHMARKS                                           │
│  • Powers: Search, Photos, Maps (1B+ users each)            │
│  • Inference: Billions of queries per day                   │
│  • Training: Multi-month runs on thousands of TPUs          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 9.4 xAI (Grok)

```
┌─────────────────────────────────────────────────────────────┐
│                      xAI INFRASTRUCTURE                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  COLOSSUS SUPERCOMPUTER (Memphis, Tennessee)                │
│  ══════════════════════════════════════════                 │
│  • 200,000+ NVIDIA H100/H200 GPUs                           │
│  • Built in 122 days (typically takes years)                │
│  • NVIDIA CEO called it "superhuman" achievement            │
│                                                             │
│  NETWORKING                                                 │
│  • NVIDIA Spectrum-X Ethernet                               │
│  • 800 Gb/s switches                                        │
│  • 95% data throughput                                      │
│  • Zero packet loss during training                         │
│                                                             │
│  POWER INFRASTRUCTURE                                       │
│  • Current consumption: 250-300 MW                          │
│  • 168 Tesla Megapacks for battery backup                   │
│                                                             │
│  GROK MODEL ARCHITECTURE                                    │
│  • Mixture-of-Experts (MoE) Transformer                     │
│  • Multi-trillion total parameters                          │
│  • Only fraction active per inference pass                  │
│                                                             │
│  EXPANSION PLANS                                            │
│  • $20B+ Mississippi facility                               │
│  • Target: 1 million GPUs by end of 2025                    │
│  • Grok 3: 10x compute of Grok-2                            │
│  • Grok 4/4 Heavy: Released July 2025                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 9.5 Meta (Llama)

```
┌─────────────────────────────────────────────────────────────┐
│                     META AI INFRASTRUCTURE                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  GPU CLUSTERS (Llama 3 Training)                            │
│  • 2 × 24,576 H100 GPU clusters (49,152 total)              │
│  • Target: 350,000 H100s by end of 2024                     │
│  • Equivalent compute: ~600,000 H100s                       │
│                                                             │
│  DUAL NETWORK ARCHITECTURE                                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Cluster 1: RoCE Fabric                               │  │
│  │  • Arista 7800 switches                               │  │
│  │  • Wedge400 + Minipack2 OCP rack switches             │  │
│  │  • 400 Gbps endpoints                                 │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Cluster 2: InfiniBand Fabric                         │  │
│  │  • NVIDIA Quantum2 InfiniBand                         │  │
│  │  • 400 Gbps endpoints                                 │  │
│  │  • Lower latency for tightly-coupled workloads        │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  OPEN HARDWARE: GRAND TETON                                 │
│  • Meta's in-house GPU server platform                      │
│  • Contributed to Open Compute Project (OCP)                │
│  • Built on OpenRack standard                               │
│                                                             │
│  LLAMA 4 TRAINING (2025)                                    │
│  • Cluster: 100,000+ H100 GPUs                              │
│  • Focus: Multimodal, longer context                        │
│  • High-bandwidth memory + low-latency interconnects        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 9.6 Infrastructure Comparison Matrix

| Company | Primary Cloud | GPUs (2025) | Custom Silicon | Open Source |
|---------|--------------|-------------|----------------|-------------|
| OpenAI | Azure + Oracle | H200, GB200 | No | No |
| Anthropic | AWS + GCP | H100, Trainium2 | No (uses AWS) | No |
| Google | GCP (owned) | - | TPU v6/v7/v8 | Partial |
| xAI | Self-hosted | 200K+ H100 | No | Yes (weights) |
| Meta | Self-hosted | 350K+ H100 | No | Yes (Llama) |

---

## 10. Safety, Alignment, and Testing

### 10.1 The RLHF Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│            REINFORCEMENT LEARNING FROM HUMAN FEEDBACK       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  STEP 1: COLLECT COMPARISON DATA                            │
│  ─────────────────────────────────                          │
│  Prompt: "How do I pick a lock?"                            │
│                                                             │
│  Response A: "Here are the steps to pick a lock..."         │
│  Response B: "I can't help with that as it could..."        │
│                                                             │
│  Human labeler: B > A (B is preferred)                      │
│                                                             │
│  STEP 2: TRAIN REWARD MODEL                                 │
│  ──────────────────────────────                             │
│  • Learn to predict human preferences                       │
│  • Input: (prompt, response) → Output: scalar score         │
│  • Higher score = more aligned with human values            │
│                                                             │
│  STEP 3: OPTIMIZE POLICY WITH PPO/DPO                       │
│  ────────────────────────────────────                       │
│  • Generate responses from current policy                   │
│  • Score with reward model                                  │
│  • Update policy to maximize reward                         │
│  • KL penalty to stay close to base model                   │
│                                                             │
│  CHALLENGE: HELPFULNESS vs HARMLESSNESS                     │
│  ───────────────────────────────────────                    │
│  • Too helpful → unsafe responses                           │
│  • Too safe → useless "I can't help" responses              │
│  • Safe RLHF: Separate reward + cost models                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 10.2 Safety Evaluation and Red Teaming

```
┌─────────────────────────────────────────────────────────────┐
│                    SAFETY EVALUATION STACK                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  BENCHMARK SUITES                                           │
│  • HarmBench: 18 attack methods, standardized evaluation    │
│  • BBQ: Social discrimination detection                     │
│  • SimpleSafetyTest: Basic safety checks                    │
│  • XSTest: Helpfulness/harmlessness balance                 │
│  • AnthropicRedTeam: Adversarial probing resilience         │
│                                                             │
│  RED TEAMING APPROACHES                                     │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Single-Turn Attacks                                  │  │
│  │  • Direct harmful requests                            │  │
│  │  • Jailbreak prompts                                  │  │
│  │  • Role-playing scenarios                             │  │
│  │  → Models generally robust here                       │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Multi-Turn Attacks (THE REAL CHALLENGE)              │  │
│  │  • Distribute harmful intent across 5-20 turns        │  │
│  │  • Build rapport, then exploit                        │  │
│  │  • 75% failure rate with persistent attacks           │  │
│  │  → Active area of research                            │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  OPENAI-ANTHROPIC JOINT EVALUATION (2025)                   │
│  • First reciprocal safety testing between competitors      │
│  • OpenAI tested: Claude Opus 4, Claude Sonnet 4            │
│  • Anthropic tested: GPT-4o, GPT-4.1, o3, o4-mini           │
│  • Finding: Multi-attempt RL attacks most effective         │
│                                                             │
│  DOCUMENTATION DEPTH                                        │
│  • Anthropic Claude Opus 4.5 system card: 153 pages         │
│  • OpenAI GPT-5 system card: 60 pages                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 10.3 Known RLHF Limitations

| Issue | Description | Mitigation |
|-------|-------------|------------|
| **Sycophancy** | Model agrees with user to maximize reward | Constitutional AI, debate |
| **Reward hacking** | Gaming the reward function | Multiple reward models |
| **Distribution shift** | Training ≠ deployment distribution | Continuous evaluation |
| **Annotation bias** | Human labelers have biases | Diverse labeler pools |
| **Helpfulness tax** | Safety reduces capability | Separate reward/cost models |

---

## 11. The Future of AI Infrastructure

### 11.1 Emerging Trends

```
┌─────────────────────────────────────────────────────────────┐
│                    INFRASTRUCTURE TRENDS                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  HARDWARE                                                   │
│  • Optical interconnects replacing copper                   │
│  • Chiplet architectures (multi-die GPUs)                   │
│  • Custom inference chips (TPU 8i, Inferentia)              │
│  • Liquid cooling becoming standard                         │
│  • Power: 1MW+ per rack becoming common                     │
│                                                             │
│  NETWORKING                                                 │
│  • 1.6 Tbps links on roadmap                                │
│  • Optical switches in backbone                             │
│  • AI-aware load balancing                                  │
│                                                             │
│  SOFTWARE                                                   │
│  • Unified training/inference frameworks                    │
│  • Better quantization (FP4, microscaling)                  │
│  • Disaggregated KV cache                                   │
│  • Speculative decoding improvements                        │
│                                                             │
│  ARCHITECTURE                                               │
│  • Mixture of Experts (MoE) becoming default                │
│  • Hybrid reasoning models (fast + deep)                    │
│  • Multi-modal native (not retrofitted)                     │
│  • Longer context (1M+ tokens)                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 11.2 Scale Projections

| Year | Largest Training Cluster | Power Consumption | Cost |
|------|-------------------------|-------------------|------|
| 2023 | ~25,000 GPUs | ~25 MW | $100M |
| 2024 | ~100,000 GPUs | ~100 MW | $500M |
| 2025 | ~500,000 GPUs | ~500 MW | $2B |
| 2026 | ~1,000,000 GPUs | ~1 GW | $5B+ |

### 11.3 Key Challenges Ahead

1. **Power and Sustainability**: AI datacenters consuming city-scale power
2. **Supply Chain**: GPU allocation as strategic resource
3. **Inference Efficiency**: Serving billions of users cost-effectively
4. **Safety at Scale**: Ensuring alignment as capabilities grow
5. **Regulatory Compliance**: Varying requirements across jurisdictions

---

## 12. Deep Technical Internals

> **This section dives into the implementation details that power modern AI systems - the algorithms, memory layouts, and architectural decisions that make inference at scale possible.**

### 12.1 FlashAttention: How GPU Memory Hierarchy Changes Everything

The attention mechanism is the computational heart of transformers, but it has a fundamental problem: it requires O(N²) memory for sequence length N. For a 32K context window, this means storing a 32,768 × 32,768 attention matrix - over 4 billion floating point numbers, or 8GB just for one attention computation. This doesn't fit in GPU SRAM, forcing constant trips to slow HBM memory.

FlashAttention, developed by Tri Dao at Stanford, solves this by exploiting the GPU memory hierarchy. The key insight is that GPUs have three levels of memory with vastly different speeds: HBM (High Bandwidth Memory) at 3.35 TB/s on H100, L2 cache at ~12 TB/s, and SRAM at ~19 TB/s. The problem is that HBM has 80GB capacity while SRAM has only 20MB - but SRAM is 5-6x faster.

The algorithm works by processing attention in tiles that fit entirely in SRAM. Instead of computing the full N×N attention matrix, FlashAttention loads small blocks of queries, keys, and values into SRAM, computes partial attention scores, and maintains running statistics to eventually produce the correct output. The mathematical trick is "online softmax" - a way to compute softmax incrementally without seeing all values first. The algorithm maintains two accumulators: the running maximum seen so far, and the running sum of exponentials. When processing a new block, it rescales previous computations based on the new maximum, ensuring numerical stability.

The performance impact is dramatic. On H100, FlashAttention-2 achieves 230 TFLOPS for attention computation compared to 85 TFLOPS for standard PyTorch attention - a 2.7x speedup. More importantly, it reduces memory usage from O(N²) to O(N), enabling context windows of 128K+ tokens that would otherwise be impossible. For a 70B model with 32K context, standard attention requires ~120GB for the attention matrices alone; FlashAttention needs only ~2GB.

The technique is now standard across all major inference frameworks. vLLM, TensorRT-LLM, and TGI all use FlashAttention variants. The original implementation is open source at github.com/Dao-AILab/flash-attention, and NVIDIA has integrated optimized versions into their cuDNN library.

```
+------------------------------------------------------------------+
|          FLASH ATTENTION PERFORMANCE (H100 SXM)                  |
+------------------------------------------------------------------+
| Sequence Length |  Standard   | FlashAttention-2 |   Speedup     |
+------------------------------------------------------------------+
|        512      |   15 ms     |      4 ms        |     3.8x      |
|      2,048      |  180 ms     |     20 ms        |     9.0x      |
|      8,192      |    OOM      |     95 ms        |     N/A       |
|     32,768      |    OOM      |    400 ms        |     N/A       |
+------------------------------------------------------------------+
| OOM = Out of Memory (standard attention can't fit in 80GB HBM)   |
+------------------------------------------------------------------+
```

### 12.2 PagedAttention: Virtual Memory Meets KV Cache

The KV cache stores the key and value tensors from previous tokens so they don't need to be recomputed. For a 70B model with 80 layers, 64 attention heads, and 128-dimensional head size, each token requires 2 × 80 × 64 × 128 × 2 = 2.6MB of KV cache. A 4K context conversation uses 10GB just for KV cache.

The traditional approach pre-allocates a contiguous memory block for the maximum possible sequence length per request. This leads to massive waste: if you allocate for 4K tokens but the actual response is 200 tokens, you've wasted 95% of that memory. Across many concurrent requests, this internal fragmentation typically wastes 60-80% of GPU memory.

PagedAttention, introduced in the vLLM paper, applies operating system virtual memory concepts to KV cache management. Instead of contiguous allocation, memory is divided into fixed-size blocks (typically 16 tokens each). Each request has a "block table" that maps logical block indices to physical memory locations - exactly like an OS page table mapping virtual to physical addresses.

When a request generates tokens, blocks are allocated on-demand from a free list. When a request completes, its blocks return to the free list immediately. There's no fragmentation because every block is the same size and blocks don't need to be contiguous. This simple change increases effective GPU memory utilization from ~20% to ~90%, enabling 3-4x more concurrent requests on the same hardware.

PagedAttention also enables "prefix caching" through copy-on-write semantics. When multiple requests share the same system prompt, they can share the same physical KV cache blocks until their outputs diverge. This is particularly valuable for applications where all requests start with the same instructions - the system prompt's KV cache is computed once and shared across thousands of requests.

The implementation requires modified attention kernels that read from non-contiguous memory. Instead of a single pointer offset, the kernel must consult the block table, look up each physical block, and gather the correct key-value pairs. This adds complexity but the memory savings far outweigh the minimal overhead.

```
+------------------------------------------------------------------+
|                   MEMORY EFFICIENCY COMPARISON                    |
+------------------------------------------------------------------+
|                                                                   |
|  Traditional Allocation (pre-allocate max length):                |
|  +--------------------+------+--------------------+------+        |
|  | Request 1 KV 2.3GB | XXXX | Request 2 KV 1.8GB | XXXX |        |
|  +--------------------+------+--------------------+------+        |
|           |              |              |            |            |
|       allocated      wasted        allocated     wasted           |
|                                                                   |
|  Efficiency: ~25% (allocate 8GB, use 2GB)                         |
|                                                                   |
|  PagedAttention (block allocation):                               |
|  +----+----+----+----+----+----+----+----+----+----+----+         |
|  | R1 | R2 | R1 | R3 | R2 | R1 | R3 | R2 | R3 | R1 |free|         |
|  +----+----+----+----+----+----+----+----+----+----+----+         |
|                                                                   |
|  Efficiency: ~95% (allocate what you need, when you need it)      |
|                                                                   |
|  Result: 3-4x more concurrent requests on same GPU                |
|                                                                   |
+------------------------------------------------------------------+
```

For implementation details, see the vLLM paper "Efficient Memory Management for Large Language Model Serving with PagedAttention" and the source at github.com/vllm-project/vllm.

### 12.3 Tensor Parallelism: Splitting Layers Across GPUs

A 70B parameter model requires 140GB in FP16 just for weights, exceeding single GPU capacity. Tensor parallelism solves this by splitting each layer across multiple GPUs, with each GPU holding a fraction of every layer's weights.

The key insight is that matrix multiplications can be partitioned. For a linear layer Y = XW, if we split W into columns [W₁|W₂|W₃|W₄] across 4 GPUs, each GPU computes Y_i = X × W_i independently. The outputs are then concatenated: Y = [Y₁|Y₂|Y₃|Y₄]. This "column parallelism" requires no communication during the forward pass - each GPU works independently.

The challenge comes with the next layer. If the first layer's output is partitioned, the second layer's input is partitioned. For row-partitioned computation, Y = X₁W₁ + X₂W₂ + X₃W₃ + X₄W₄, we need to sum the partial results across GPUs. This requires an "all-reduce" collective communication operation.

The standard pattern for transformer MLPs is: Column Parallel (first linear) → Activation → Row Parallel (second linear). This minimizes communication to one all-reduce per MLP block. Attention layers use similar patterns, with query/key/value projections column-parallel and output projection row-parallel.

The all-reduce operation uses a ring algorithm where each GPU sends data to its neighbor in a ring topology. For N GPUs with data size D, each GPU sends and receives 2×(N-1)/N × D bytes total. With NVLink providing 1.8 TB/s bidirectional bandwidth between GPUs, a 100MB tensor all-reduce across 8 GPUs takes under 1ms.

```
+------------------------------------------------------------------+
|                   TENSOR PARALLELISM LAYOUT                       |
+------------------------------------------------------------------+
|                                                                   |
|  MLP Layer Pattern:                                               |
|                                                                   |
|  Input X (replicated on all GPUs)                                 |
|       |                                                           |
|       v                                                           |
|  +----+----+----+----+                                            |
|  |Col |Col |Col |Col |   First Linear (Column Parallel)           |
|  | 0  | 1  | 2  | 3  |   No communication needed                  |
|  +----+----+----+----+                                            |
|       |                                                           |
|       v                                                           |
|  Activation (GELU) - local                                        |
|       |                                                           |
|       v                                                           |
|  +----+----+----+----+                                            |
|  |Row |Row |Row |Row |   Second Linear (Row Parallel)             |
|  | 0  | 1  | 2  | 3  |   Partial sums computed locally            |
|  +----+----+----+----+                                            |
|       |                                                           |
|       v                                                           |
|  [=== ALL-REDUCE ===]   Sum partial results across GPUs           |
|       |                                                           |
|       v                                                           |
|  Output Y (replicated on all GPUs)                                |
|                                                                   |
|  Communication: 1 all-reduce per MLP block                        |
|  With 8 GPUs on NVLink: ~0.5ms per layer                          |
|                                                                   |
+------------------------------------------------------------------+
```

For implementation, NVIDIA's Megatron-LM (github.com/NVIDIA/Megatron-LM) is the reference. It's used by most companies training large models and includes optimized tensor parallel primitives.

### 12.4 FP8 Quantization: Half the Bits, Double the Speed

H100 and B200 GPUs include dedicated Tensor Cores for FP8 (8-bit floating point) computation, offering 2x the FLOPS of FP16/BF16. NVIDIA's Transformer Engine library enables FP8 training and inference with minimal accuracy loss through careful scaling.

FP8 comes in two formats: E4M3 (4 exponent bits, 3 mantissa bits) with range ±448, and E5M2 (5 exponent bits, 2 mantissa bits) with range ±57,344. E4M3 offers more precision and is used for weights and activations during forward passes. E5M2 provides more range and handles gradients during backpropagation, which can have larger outlier values.

The challenge is that 8 bits provide very limited dynamic range. A BF16 tensor with values spanning 0.001 to 1000 can't be directly cast to FP8 without massive precision loss. The solution is per-tensor scaling: multiply all values by a scale factor that maps the tensor's range into FP8's representable range, perform computation in FP8, then divide by the scale.

The clever innovation is "delayed scaling." Computing the optimal scale requires finding the tensor's maximum absolute value, which is expensive. Delayed scaling uses the scale computed from the previous training iteration, based on the assumption that tensor distributions don't change dramatically between iterations. This eliminates the scaling overhead while maintaining accuracy.

In practice, FP8 training achieves within 0.1% of BF16 perplexity on language models. The memory savings come from storing weights and activations in 8 bits instead of 16, halving memory requirements. Combined with 2x Tensor Core throughput, FP8 delivers roughly 2x end-to-end training speedup.

```
+------------------------------------------------------------------+
|                    FP8 FORMAT COMPARISON                          |
+------------------------------------------------------------------+
|                                                                   |
|  E4M3 (for forward pass):                                         |
|  +---+--------+-------+                                           |
|  | S | E (4b) | M (3b)|    Range: ±448                            |
|  +---+--------+-------+    Precision: 8 distinct mantissa values  |
|  |                         Best for: Weights, activations         |
|  |                                                                |
|  E5M2 (for backward pass):                                        |
|  +---+----------+-----+                                           |
|  | S |  E (5b)  |M(2b)|    Range: ±57,344                         |
|  +---+----------+-----+    Precision: 4 distinct mantissa values  |
|                            Best for: Gradients (can have outliers)|
|                                                                   |
+------------------------------------------------------------------+
|                                                                   |
|  PERFORMANCE COMPARISON (H100):                                   |
|                                                                   |
|  Operation       |   BF16   |   FP8    |  Speedup                 |
|  ----------------+----------+----------+-----------               |
|  GEMM TFLOPS     |  1,979   |  3,958   |   2.0x                   |
|  Memory BW       |  3.35TB/s|  3.35TB/s|   1.0x                   |
|  Effective BW*   |  3.35TB/s|  6.70TB/s|   2.0x                   |
|                                                                   |
|  * Half the bytes means twice the effective data throughput       |
|                                                                   |
+------------------------------------------------------------------+
```

NVIDIA's Transformer Engine (github.com/NVIDIA/TransformerEngine) provides drop-in FP8 support for PyTorch models.

### 12.5 Speculative Decoding: Parallelizing the Unparallelizable

Autoregressive decoding is inherently sequential: each token depends on all previous tokens. With large models taking 30-50ms per token, generating a 500-token response takes 15-25 seconds. Speculative decoding breaks this barrier by using a small "draft" model to propose multiple tokens that the large "target" model verifies in parallel.

The algorithm works as follows: A draft model (e.g., 7B parameters) quickly generates K candidate tokens (typically K=4-5). These candidates are then fed to the target model (e.g., 70B) as a single forward pass, which computes the probability of each token given all preceding tokens. Through rejection sampling, we accept tokens where the draft model's predictions match the target model's distribution, and reject where they diverge.

The mathematical guarantee is crucial: the output distribution is identical to sampling directly from the target model. This isn't an approximation - through careful probability adjustment when rejecting tokens, speculative decoding produces exactly the same distribution of outputs as standard decoding, just faster.

The speedup depends on the acceptance rate - how often the draft model's predictions match the target model's. For well-aligned models (same training data, same architecture family), acceptance rates of 70-80% are common. This means 3-4 tokens accepted per verification pass on average, yielding 1.5-2x speedup for latency-sensitive single-request scenarios.

The technique is most valuable for interactive applications where latency matters more than throughput. For batch processing with many concurrent requests, standard batching already achieves high GPU utilization, and speculative decoding's benefits diminish.

An alternative approach is Medusa, which attaches multiple small "speculation heads" directly to the target model rather than using a separate draft model. Each head predicts a different future token position (+1, +2, +3, etc.). This eliminates the need to load a separate draft model into memory and allows the speculation heads to leverage the target model's hidden states directly.

```
+------------------------------------------------------------------+
|                 SPECULATIVE DECODING FLOW                         |
+------------------------------------------------------------------+
|                                                                   |
|  Standard decoding (sequential):                                  |
|                                                                   |
|  [70B]-->[70B]-->[70B]-->[70B]-->[70B]                            |
|   50ms   50ms    50ms    50ms    50ms  = 250ms for 5 tokens       |
|                                                                   |
|  Speculative decoding:                                            |
|                                                                   |
|  Step 1: Draft generates candidates quickly                       |
|  [7B]->[7B]->[7B]->[7B] = 20ms total                              |
|  Candidates: "The", "quick", "brown", "fox"                       |
|                                                                   |
|  Step 2: Target verifies ALL in one forward pass                  |
|  [========== 70B verify all 4 ===========] = 55ms                 |
|                                                                   |
|  Step 3: Accept/reject via probability comparison                 |
|  "The" -> Accept (p_target > p_draft)                             |
|  "quick" -> Accept                                                |
|  "brown" -> Reject (resample from target)                         |
|  "fox" -> Not evaluated (prior token rejected)                    |
|                                                                   |
|  Result: 3 tokens in 75ms instead of 150ms = 2x speedup           |
|                                                                   |
+------------------------------------------------------------------+
|                                                                   |
|  ACCEPTANCE RATES BY TASK TYPE:                                   |
|                                                                   |
|  Task              | Accept Rate | Effective Speedup              |
|  ------------------+-------------+-------------------             |
|  Code generation   |    72%      |      1.8x                      |
|  Q&A (factual)     |    78%      |      2.0x                      |
|  Creative writing  |    65%      |      1.6x                      |
|  Math/reasoning    |    52%      |      1.4x                      |
|                                                                   |
+------------------------------------------------------------------+
```

### 12.6 Continuous Batching: Why vLLM is 10x Faster

Traditional batching waits for a fixed batch to fill, processes all requests together, then waits for the slowest request to finish before starting the next batch. This creates two sources of waste: waiting time at batch boundaries, and padding shorter sequences to match the longest.

Continuous batching (also called "iteration-level batching") eliminates both. The scheduler runs a tight loop: every 20-30ms, it decides which requests to process next. When a request completes (generates end-of-sequence), its slot immediately becomes available for a waiting request. There's no waiting for batch boundaries.

The vLLM scheduler maintains three queues: waiting (new requests), running (actively generating), and swapped (temporarily moved to CPU memory). Each iteration, the scheduler:

1. Tries to swap in previously evicted requests from CPU
2. Admits new requests from the waiting queue if memory allows
3. Allocates KV cache blocks for running requests
4. If memory is exhausted, preempts the most recently started request (LIFO policy, since newer requests have less compute invested)

The preemption mechanism is crucial for handling memory pressure gracefully. Instead of rejecting requests or crashing when GPU memory fills, the scheduler can temporarily evict requests to CPU memory (preserving their KV cache) or force them to restart (discarding KV cache). This enables the system to handle burst traffic while maintaining responsiveness.

Prefill (processing the input prompt) and decode (generating output tokens) have very different compute characteristics. Prefill processes many tokens in parallel and is compute-bound. Decode processes one token per sequence and is memory-bandwidth-bound. vLLM handles both in the same batch by separating them into different code paths within a single forward pass.

The result is dramatic: vLLM achieves 2-24x higher throughput than HuggingFace TGI depending on workload characteristics. The improvement is largest for high request rates with variable-length outputs, where traditional batching wastes the most resources on padding and synchronization.

```
+------------------------------------------------------------------+
|              BATCHING STRATEGY COMPARISON                         |
+------------------------------------------------------------------+
|                                                                   |
|  Static Batching:                                                 |
|  +------------------+------+                                      |
|  | Request A (100t) | XXXX | <- waits for A to finish             |
|  +------------------+------+                                      |
|  | Request B (50t)  | XXXXXXXX | <- padded, wasted compute        |
|  +--------------+---+----------+                                  |
|  | Request C (75t)  | XXXXX |                                     |
|  +------------------+-------+                                     |
|                              ^                                    |
|                          batch boundary                           |
|                              |                                    |
|  +------------------+                                             |
|  | Request D starts | <- must wait for entire batch to finish     |
|  +------------------+                                             |
|                                                                   |
|  Continuous Batching:                                             |
|  +------------------+                                             |
|  | Request A (100t) |                                             |
|  +------------------+--------+                                    |
|  | Request B (50t) | Req D.. | <- D joins immediately when B ends |
|  +--------------+--+---------+---+                                |
|  | Request C (75t) |  Req E....  | <- E joins when C ends         |
|  +------------------+------------+                                |
|                                                                   |
|  No wasted padding, no waiting at boundaries                      |
|  Result: 2-24x higher throughput                                  |
|                                                                   |
+------------------------------------------------------------------+
```

### 12.7 Distributed Checkpointing: Saving Terabytes in Seconds

Training a 70B model produces over 1TB of state that must be checkpointed: 280GB of model weights, 280GB of optimizer momentum, 280GB of optimizer variance, plus gradients and random states. With training runs lasting months and costing millions of dollars, losing progress to a hardware failure is unacceptable.

The naive approach - gather all state to rank 0 and write to disk - doesn't scale. Gathering 1TB to a single node overwhelms memory, and writing serially takes hours. Modern distributed checkpointing solves this by having each GPU write its shard directly to a parallel filesystem.

With FSDP (Fully Sharded Data Parallelism) or DeepSpeed ZeRO-3, each GPU holds only 1/N of the model state for N GPUs. A 70B model across 1024 GPUs means ~1GB per GPU. Each GPU writes its shard independently to a different file, achieving aggregate write bandwidth of hundreds of GB/s across the parallel filesystem.

PyTorch's Distributed Checkpoint (DCP) format stores each tensor in a separate file with metadata describing the global tensor shapes and how they're partitioned. This enables "resharding" - loading a checkpoint saved with 1024 GPUs onto 512 GPUs, or vice versa. The loader reads the metadata, determines which portions of each tensor it needs, and loads only those portions.

Async checkpointing overlaps saving with training. The trainer initiates a checkpoint, continues to the next training iteration, and the I/O completes in the background. This hides checkpoint latency entirely for typical configurations.

```
+------------------------------------------------------------------+
|                 CHECKPOINT ANATOMY (70B Model)                    |
+------------------------------------------------------------------+
|                                                                   |
|  Component              | Size (FP32)  | Size (BF16)              |
|  -----------------------+--------------+-------------             |
|  Model weights          |    280 GB    |    140 GB                |
|  Optimizer momentum (m) |    280 GB    |    280 GB                |
|  Optimizer variance (v) |    280 GB    |    280 GB                |
|  Gradients (optional)   |    280 GB    |    140 GB                |
|  RNG states             |     ~1 GB    |     ~1 GB                |
|  -----------------------+--------------+-------------             |
|  TOTAL                  |  ~1.12 TB    |   ~841 GB                |
|                                                                   |
+------------------------------------------------------------------+
|                                                                   |
|  FILESYSTEM PERFORMANCE FOR 1TB CHECKPOINT:                       |
|                                                                   |
|  Filesystem        | Write BW   | Time to Save                    |
|  ------------------+------------+---------------                  |
|  Local NVMe SSD    | 7 GB/s     | 2.4 minutes                     |
|  Lustre (parallel) | 100+ GB/s  | 10 seconds                      |
|  GPFS/Spectrum     | 80+ GB/s   | 12 seconds                      |
|  WekaFS            | 150+ GB/s  | 7 seconds                       |
|                                                                   |
+------------------------------------------------------------------+
```

### 12.8 Mixture of Experts: Trillion Parameters, Fraction of Compute

Mixture of Experts (MoE) architectures enable trillion-parameter models with the compute cost of much smaller dense models. GPT-4, Mixtral, and Grok all use MoE.

The concept is simple: instead of one large feed-forward network in each transformer layer, have N "expert" networks (typically 8-64). A learned "router" network examines each token and selects the top-K experts (usually K=1 or K=2) to process that token. Different tokens activate different experts.

For a model with 8 experts and top-2 routing, each token uses only 2/8 = 25% of the expert parameters. A 1.8 trillion parameter MoE model with 8 experts uses roughly the same compute per token as a 225B dense model, but has access to much more total knowledge stored in the inactive experts.

The routing decision is learned end-to-end with the model. During training, gradients flow through the router, teaching it which experts are best suited for different types of content. In practice, experts tend to specialize - some handle code, others handle different languages, others handle reasoning patterns.

Load balancing is critical. If the router always selects the same expert, the other experts never learn and GPU utilization becomes uneven. MoE training includes auxiliary losses that encourage balanced expert usage. Token dropping (discarding tokens routed to overloaded experts) and expert capacity limits ensure even distribution.

```
+------------------------------------------------------------------+
|                    MIXTURE OF EXPERTS LAYER                       |
+------------------------------------------------------------------+
|                                                                   |
|  Input Token Embedding                                            |
|          |                                                        |
|          v                                                        |
|    +----------+                                                   |
|    |  Router  |  (small MLP: hidden_size -> num_experts)          |
|    +----------+                                                   |
|          |                                                        |
|          v                                                        |
|    [0.1, 0.05, 0.7, 0.02, 0.08, 0.02, 0.02, 0.01]  <- softmax     |
|                    ^                                              |
|              Expert 2 wins (0.7)                                  |
|                    |                                              |
|          +---------+---------+                                    |
|          |         |         |                                    |
|          v         v         v                                    |
|     +--------+ +--------+ +--------+                              |
|     |Expert 0| |Expert 2| |Expert 7|   (8 total experts)          |
|     | (idle) | |(active)| | (idle) |                              |
|     +--------+ +--------+ +--------+                              |
|                    |                                              |
|                    v                                              |
|              Output Token                                         |
|                                                                   |
|  Result: 8x parameters, ~1x compute per token                     |
|                                                                   |
+------------------------------------------------------------------+
|                                                                   |
|  REAL-WORLD MOE CONFIGURATIONS:                                   |
|                                                                   |
|  Model       | Total Params | Experts | Top-K | Active Params     |
|  ------------+--------------+---------+-------+--------------     |
|  Mixtral 8x7B|    47B       |    8    |   2   |    13B            |
|  GPT-4 (est) |   1.8T       |   16    |   2   |   ~220B           |
|  Grok-1      |   314B       |    8    |   2   |    ~86B           |
|                                                                   |
+------------------------------------------------------------------+
```

### 12.9 KV Cache Memory Economics

Understanding KV cache memory requirements is essential for capacity planning. The formula is: KV_size = 2 × num_layers × num_kv_heads × head_dim × seq_len × bytes_per_value.

For LLaMA 70B: 2 × 80 layers × 8 KV heads (GQA) × 128 dim × 4096 tokens × 2 bytes (FP16) = 2.6GB per request. With 80GB HBM and 50GB for model weights, you have 30GB for KV cache, supporting ~11 concurrent requests at 4K context.

Grouped Query Attention (GQA) significantly reduces this. Instead of separate key-value heads for each attention head (64 heads = 64 KV heads), GQA shares KV heads across groups. LLaMA 70B uses 8 KV heads shared across 64 attention heads, an 8x reduction in KV cache size compared to standard multi-head attention.

Context length has dramatic impact. Doubling context from 4K to 8K doubles KV cache per request and halves maximum concurrency. This is why H200 with 141GB HBM (vs H100's 80GB) is so valuable for inference - more memory means more concurrent long-context requests.

```
+------------------------------------------------------------------+
|                 KV CACHE SIZING EXAMPLES                          |
+------------------------------------------------------------------+
|                                                                   |
|  Model        | Layers | KV Heads | Head Dim | Per-Token KV       |
|  -------------+--------+----------+----------+--------------      |
|  LLaMA 7B     |   32   |    32    |   128    |    512 KB          |
|  LLaMA 70B    |   80   |     8    |   128    |    640 KB          |
|  GPT-4 (est)  |  120   |    16    |   128    |   1.2 MB           |
|                                                                   |
|  AT 4K CONTEXT:                                                   |
|  LLaMA 70B: 640 KB x 4096 = 2.6 GB per request                    |
|                                                                   |
|  GPU Memory Budget (H100 80GB):                                   |
|  - Model weights:    50 GB                                        |
|  - CUDA overhead:     5 GB                                        |
|  - KV Cache:         25 GB (available)                            |
|  - Max concurrent:   25 GB / 2.6 GB = ~9 requests at 4K context   |
|                                                                   |
|  With H200 (141GB):                                               |
|  - KV Cache:         86 GB (available)                            |
|  - Max concurrent:   86 GB / 2.6 GB = ~33 requests at 4K context  |
|                                                                   |
+------------------------------------------------------------------+
```

### 12.10 The Prefill-Decode Dichotomy

Inference has two distinct phases with opposite computational characteristics.

**Prefill** (processing the input prompt) is compute-bound. All input tokens are processed in parallel through matrix multiplications. GPU Tensor Cores are fully utilized. Latency scales sub-linearly with prompt length due to parallelization. A 4K token prompt might take 200ms.

**Decode** (generating output tokens) is memory-bandwidth-bound. Each iteration generates exactly one new token, requiring full model weights to be read from HBM for a tiny amount of computation. Tensor Cores are idle most of the time waiting for memory. Latency is roughly constant per token regardless of context length - about 20-50ms per token for large models.

This dichotomy explains why H200's increased memory bandwidth (4.8 TB/s vs H100's 3.35 TB/s) matters more than its compute capability for inference. Decode is waiting on memory, not compute.

Chunked prefill splits long prompts into chunks, interleaving with decode operations. This improves latency for decode requests that would otherwise wait behind a long prefill. GKE's Inference Gateway and vLLM both implement this.

Disaggregated prefill-decode takes this further: dedicate some GPUs to prefill-only and others to decode-only. Prefill GPUs are configured for maximum throughput (large batches, high utilization). Decode GPUs are configured for minimum latency. The KV cache is transferred between them. This is an emerging architecture pattern for large-scale deployments.

```
+------------------------------------------------------------------+
|              PREFILL VS DECODE CHARACTERISTICS                    |
+------------------------------------------------------------------+
|                                                                   |
|  Characteristic      |    Prefill        |     Decode             |
|  --------------------+-------------------+------------------      |
|  Tokens per forward  | Hundreds/thousands|     Exactly 1          |
|  Bottleneck          | Compute (TFLOPS)  |  Memory BW (TB/s)      |
|  Tensor Core usage   |     High (~80%)   |     Low (~10%)         |
|  Scaling             | Sub-linear w/len  |  Constant per token    |
|  Batching benefit    |     Moderate      |      Huge              |
|                                                                   |
|  Example (70B model, H100):                                       |
|  - Prefill 4K tokens:  200ms (50 TFLOPS utilized)                 |
|  - Decode 1 token:      30ms (memory bound)                       |
|  - Decode 500 tokens: 15 seconds                                  |
|                                                                   |
|  Optimization Focus:                                              |
|  - Prefill: Maximize batch size, FlashAttention                   |
|  - Decode: Maximize memory bandwidth, batching, speculation       |
|                                                                   |
+------------------------------------------------------------------+
```

---

## Appendix A: Glossary

| Term | Definition |
|------|------------|
| **Attention** | Mechanism allowing tokens to weight importance of other tokens |
| **Batch Size** | Number of samples processed before weight update |
| **BF16** | Brain Float 16 - 16-bit floating point format optimized for ML |
| **Continuous Batching** | Adding new requests to batch without waiting |
| **CUDA** | NVIDIA's parallel computing platform |
| **Embedding** | Dense vector representation of a token |
| **FLOPS** | Floating Point Operations Per Second |
| **HBM** | High Bandwidth Memory - stacked DRAM for GPUs |
| **Inference** | Using trained model to generate predictions |
| **KV Cache** | Stored key-value pairs from attention for reuse |
| **MoE** | Mixture of Experts - sparse model architecture |
| **NVLink** | NVIDIA's GPU interconnect technology |
| **PagedAttention** | Memory-efficient attention for variable sequences |
| **PPO** | Proximal Policy Optimization - RL algorithm |
| **Prefill** | Processing input tokens before generation |
| **Quantization** | Reducing numerical precision to save memory/compute |
| **RLHF** | Reinforcement Learning from Human Feedback |
| **RoCE** | RDMA over Converged Ethernet |
| **Tensor Core** | Specialized hardware for matrix operations |
| **TTFT** | Time to First Token |
| **Transformer** | Neural network architecture using attention |

---

## Appendix B: Further Reading

### Official Engineering Blogs
- [OpenAI Research](https://openai.com/research)
- [Anthropic Research](https://www.anthropic.com/research)
- [Google AI Blog](https://blog.google/technology/ai/)
- [Meta AI Blog](https://ai.meta.com/blog/)
- [NVIDIA Technical Blog](https://developer.nvidia.com/blog/)

### Key Papers
- "Attention Is All You Need" (Vaswani et al., 2017)
- "Training language models to follow instructions with human feedback" (OpenAI, 2022)
- "Constitutional AI: Harmlessness from AI Feedback" (Anthropic, 2022)
- "LLaMA: Open and Efficient Foundation Language Models" (Meta, 2023)

### Infrastructure Deep-Dives
- [Building Meta's GenAI Infrastructure](https://engineering.fb.com/2024/03/12/data-center-engineering/building-metas-genai-infrastructure/)
- [Google TPU Architecture Guide](https://cloud.google.com/tpu)
- [vLLM: Easy, Fast, and Cheap LLM Serving](https://vllm.ai/)

---

## Sources

This document was compiled from the following sources:

- [OpenAI - Building Compute Infrastructure for the Intelligence Age](https://openai.com/index/building-the-compute-infrastructure-for-the-intelligence-age/)
- [Anthropic SpaceX Compute Deal Analysis](https://www.mindstudio.ai/blog/anthropic-spacex-compute-deal-claude-rate-limits)
- [Google TPUs Explained: Architecture & Performance](https://intuitionlabs.ai/articles/google-tpu-architecture-gemini-3)
- [xAI Complete Guide](https://theairankings.com/xai/)
- [Meta GenAI Infrastructure](https://engineering.fb.com/2024/03/12/data-center-engineering/building-metas-genai-infrastructure/)
- [NVIDIA H100 vs H200 vs B200 Comparison](https://introl.com/blog/h100-vs-h200-vs-b200-choosing-the-right-nvidia-gpus-for-your-ai-workload)
- [LLM Inference Optimization Guide 2025](https://www.youngju.dev/blog/culture/2026-04-14-llm-inference-optimization-vllm-trt-llm-serving-guide-2025.en)
- [Kubernetes Autoscaling for LLM Inference](https://collabnix.com/kubernetes-autoscaling-for-llm-inference-complete-guide-2024/)
- [OpenAI-Anthropic Safety Evaluation](https://openai.com/index/openai-anthropic-safety-evaluation/)
- [Transformer Architecture - Wikipedia](https://en.wikipedia.org/wiki/Transformer_(deep_learning))
- [InfiniBand vs RoCE for AI Networks](https://www.vitextech.com/blogs/blog/infiniband-vs-ethernet-for-ai-clusters-effective-gpu-networks-in-2025)
