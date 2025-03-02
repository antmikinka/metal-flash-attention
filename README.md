# Metal FlashAttention

A faster alternative to Metal Performance Shaders, a reference implementation of modern GPU algorithms, and a step toward defragmenting the AI ecosystem.

Algorithms:
- [x] Attention
  - [x] Dense (90.5% ALU)
  - [x] Block-Sparse
- [x] GEMM
  - [x] FP16 (93.3% ALU)
  - [x] FP32 (87.2% ALU)
    - [ ] Complex Numbers
    - [ ] FP64 Accumulate
  - [x] Fused Biases
- [ ] Linear Algebra
  - [ ] Cholesky
  - [ ] Triangular Solve

## Usage

| Progamming Language | MFA Supports | MPSGraph Supports | PyTorch Supports |
| ------------------- | ------------ | ------------ | ---------------- |
| CPU C++ (metal-cpp)                | ✅ | ❌ | ✅ |
| GPU C++ (Indirect Command Buffers) | ✅ | ❌ | ❌ | 
| Swift (iPadOS, Playgrounds)        | ✅ | ✅ | ❌ |
| Swift (macOS, Xcode)               | ✅ | ✅ | ✅ |
| Predecessor to Swift       | not tested | ✅ | ✅ |

Usage:
- Download Xcode 14.2 from the Apple [developer tools archive](https://developer.apple.com/download/all/?q=xcode)
- Run the Swift script to compile `libMetalFlashAttention.metallib`
- Read the [API specification](./Documentation/API.md)
- Generate Metal shader variants at runtime

Alternatively:
- Download the newest version of Xcode
- Fetch the Metal library from [GitHub releases](https://github.com/philipturner/metal-flash-attention/releases)
- Run the unit tests from this repository

## Performance

SGEMM, every square matrix from 1&ndash;1536:

![Max GFLOPS achieved](./Documentation/SGEMM.png)

HGEMM, every square matrix from 1&ndash;2048:

![Max GFLOPS achieved](./Documentation/HGEMM.png)

### GEMM

Scaling by square size:
- Matrix M: every even integer
- Matrix N: every even integer
- Matrix K: every even integer
- For 2x batched, every multiple of 4
- For very large square matrices, granularity varies

| Function Constant | Value |
| ------ | --------- |
| `M_splits` | 2 |
| `N_splits` | 2 |
| `M_simd` | Block M / `M_splits` |
| `N_simd` | Block N / `N_splits` |
| `K_simd` | Block K |
  
| Precision | Block M | Block N | Block K |
| - | - | - | - |
| Float32 | 32 | 32 | 32 |
| Float32 | 48 | 48 | 24 |
| Float16 | 32 | 32 | 32 |
| Float16 | 48 | 48 | 32 |

| Size Start | Size End | Duplicate Commands/Encoder | Trials |
| ---------- | -------- | ---------- | ------ |
| 1 | 190 | 256 | 16 |
| 192 | 254 | 128 | 16 |
| 256 | 382 | 64 | 16 |
| 384 | 510 | 32 | 16 |
| 512 | 766 | 16 | 16 |
| 768 | 1022 | 8 | 16 |
| 1024 | 1534 | 4 | 16 |
| 1536 | 2048 | 2 | 16 |

### Float32 Utilization (NN)
 
![Float32 Utilization (NN)](./CI/float32-nn-latest.png)

### Float32 Utilization (NT)

![Float32 Utilization (NT)](./CI/float32-nt-latest.png)

### Float32 Utilization (NT, Large)

![Float32 Utilization (NT)](./CI/float32-nt-large-latest.png)

### Float16 Utilization (NN)

![Float16 Utilization (NN)](./CI/float16-nn-latest.png)

### Float16 Utilization (NT, 2x Batched)

![Float16 Utilization (NT, 2x Batched)](./CI/float16-nt-batched-latest.png)

### Float16 Utilization (NTN, 2x Batched, Bias)

![Float16 Utilization (NTN, 2x Batched, Bias)](./CI/float16-ntn-batched-bias-latest.png)

### Attention

Setup:
- Sequence dimension:
  - R = rows (output sequence length)
  - C = columns (input sequence length)
  - R = C
- Masking:
  - Only MFA supports block-sparse masks.
  - For "scaling by sparsity", sparse block size equals GEMM block size.

Scaling by sequence length:
- Masking:
  - No mask
  - Dense Mask: triangular mask
  - Sparse Mask: triangular mask, summarized by block-sparse mask
- Sequence length:
  - Small sequences: every multiple of 4
  - Large sequences: every multiple of 64
  - Causal mask: every even integer
- Head size: 64
- Head count:
  - Small sequences: 10
  - Large sequences: 5
  - Causal mask: 10

Scaling by head size:
- Masking: dense, no mask
- Sequence length 4096
- Head size: every integer
  - &le;64: every integer
  - &gt;64: every `roundUpToPowerOf2(D/64)` integers
- Head count: 8
  
| Function Constant | Value |
| ------ | --------- |
| `Q_trans` | ❌ |
| `K_trans` | ✅ |
| `V_trans` | ❌ |
| `O_trans` | ❌ |
| `R_splits` | TBD |
| `R_simd` | Block R / `R_splits` |
| `C_simd` | Block C |
| `D_simd` | $$8 \times \left \lceil{ \frac{D}{8} }\right \rceil $$  |

### Float32 Sequence Scaling (Small)

![FlashAttention (F32, H=10, D=64)](./CI/float32-small-sequences-latest.png)

### Float16 Sequence Scaling (Small)

Dense: Stable Diffusion XL outermost attention layer @ 512x512 (sequence length = 1024)

![FlashAttention (F16, H=10, D=64)](./CI/float16-small-sequences-latest.png)

### Float16 Sequence Scaling (Large)

Dense: Stable Diffusion 2 outermost attention layer @ 512x512 (sequence length = 4096)

![FlashAttention (F16, H=5, D=64)](./CI/float16-large-sequences-latest.png)

### Float32 Sequence Scaling (Causal Mask)

![FlashAttention (F32, H=10, D=64)](./CI/float32-large-causal-latest.png)

### Float16 Sequence Scaling (Causal Mask)

![FlashAttention (F16, H=10, D=64)](./CI/float16-small-causal-latest.png)

![FlashAttention (F16, H=10, D=64)](./CI/float16-large-causal-latest.png)

### Float16 Head Scaling

Dense: Stable Diffusion 1 outermost attention layer @ 512x512 (head size = 40)

![FlashAttention (F16, R=C=4096, H=8)](./CI/float16-head-sizes-latest.png)

## Roadmap

Releases:
- v0.1.0-alpha
  - Initial release, only non-batched GEMM without fused transposes
- v0.2.0-alpha
  - Fused transposes for A and B
  - Batched GEMM
- v1.0.0
  - Attention: dense and block-sparse
- v1.0.1
  - GEMM: fused biases
