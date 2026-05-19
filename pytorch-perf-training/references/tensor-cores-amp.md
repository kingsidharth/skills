# Tensor Cores & AMP

## Enabling Tensor Cores

Tensor Cores require:
1. **Compatible dtype**: fp16, bf16, or tf32 for the matmul operands
2. **Aligned dimensions**: matrix dimensions should be multiples of 8 (fp16/bf16) or multiples of 4 (tf32)
3. **Correct backend settings**

```python
# Enable tf32 for fp32 matmuls (Ampere+) — free speedup, tiny precision loss
torch.backends.cuda.matmul.allow_tf32 = True
torch.backends.cudnn.allow_tf32 = True

# cuDNN autotuner — benchmarks convolution algorithms
torch.backends.cudnn.benchmark = True
```

tf32 uses Tensor Cores for fp32 matmuls by truncating mantissa to 10 bits. Roughly 3x faster than pure fp32 with negligible training impact.

## AMP (Automatic Mixed Precision)

### bf16 (recommended for Ampere+)

```python
# bf16 — same exponent range as fp32, no grad scaler needed
with torch.amp.autocast("cuda", dtype=torch.bfloat16):
    output = model(input)
    loss = criterion(output, target)

loss.backward()
optimizer.step()
```

### fp16 (older GPUs or when bf16 unavailable)

```python
scaler = torch.amp.GradScaler()

with torch.amp.autocast("cuda", dtype=torch.float16):
    output = model(input)
    loss = criterion(output, target)

scaler.scale(loss).backward()
scaler.step(optimizer)
scaler.update()
optimizer.zero_grad(set_to_none=True)
```

### What stays in fp32

AMP auto-casts most ops to the lower precision, but keeps these in fp32:
- Loss functions (cross-entropy, MSE)
- Softmax
- Layer normalization (internally)
- Small reductions

You can customise:
```python
# Force a specific op to stay in fp32
with torch.amp.autocast("cuda", dtype=torch.bfloat16):
    # This linear runs in bf16
    x = self.linear1(x)
    # Force fp32 for a sensitive operation
    with torch.amp.autocast("cuda", enabled=False):
        x = sensitive_op(x.float())
```

## Shape alignment for Tensor Cores

Tensor Cores operate on tiles. Misaligned dimensions waste compute:

```
Optimal: hidden_dim = 768 (divisible by 8)
Bad:     hidden_dim = 769

Optimal: batch_size = 64 (divisible by 8)
Bad:     batch_size = 63
```

For DiT models: ensure `embed_dim`, `num_heads * head_dim`, `mlp_hidden`, and `batch_size` are all multiples of 8.

torch.compile's `shape_padding` option can auto-pad:
```python
model = torch.compile(model, options={"shape_padding": True})
```

## channels_last memory format

For conv-heavy models (not typical DiT, but relevant for VAE encoder/decoder):

```python
model = model.to(memory_format=torch.channels_last)
input = input.to(memory_format=torch.channels_last)
```

Enables more efficient Tensor Core utilisation for convolutions by matching NHWC layout expected by cuDNN.

## Verifying Tensor Core usage

In a profiler trace, Tensor Core kernels have names containing:
- `sm80_xmma` (Ampere)
- `sm90_xmma` (Hopper)
- `ampere_` prefix
- `cutlass` or `cublasLt` with `_tf32` / `_f16` / `_bf16` suffix

If you see only `sgemm` (single-precision GEMM), Tensor Cores are not engaged.

## DiT-specific: AdaLN modulation

DiT's adaptive layer norm (`adaLN-Zero`) uses element-wise scale/shift from timestep embeddings. These are pointwise ops — not Tensor Core candidates, but torch.compile fuses them into surrounding kernels effectively.
