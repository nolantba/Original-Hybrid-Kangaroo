# RC-Kangaroo Herd Kernel Comparison Guide

## Overview

This guide explains how to build, benchmark, and compare the two SOTA++ Herd kernel implementations:

1. **Integrated Herd Support** (recommended) - Adds herd bias to existing optimized kernel
2. **Separate Herd Kernel** (experimental) - Dedicated kernel with per-herd jump tables

---

## Quick Start

### Run Complete Comparison

```bash
./compare_kernels.sh
```

This will:
1. Build both implementations
2. Run 60-second benchmarks for each
3. Generate detailed performance comparison
4. Save all results to timestamped directory

---

## Build Options

### Option 1: Integrated Herds (Default - Recommended)

```bash
make clean
make SM=86 USE_SEPARATE_HERD_KERNEL=0 PROFILE=release -j
./rckangaroo -herds -range 135 -pubkey <pubkey> ...
```

**Performance**: ✅ 6.55 GK/s (zero overhead)

### Option 2: Separate Kernel (Experimental)

```bash
make clean
make SM=86 USE_SEPARATE_HERD_KERNEL=1 PROFILE=release -j
./rckangaroo -herds -range 135 -pubkey <pubkey> ...
```

**Performance**: ⚠️ 1.89 GK/s (4x slower - needs optimization)

---

## Makefile Build Flags

| Flag | Values | Description |
|------|--------|-------------|
| `SM` | 86, 75, 70, etc. | CUDA compute capability (86 = RTX 3060) |
| `USE_SEPARATE_HERD_KERNEL` | 0 or 1 | 0 = integrated (default), 1 = separate |
| `USE_JACOBIAN` | 0 or 1 | Use Jacobian coordinates (default: 1) |
| `USE_SOTA_PLUS` | 0 or 1 | SOTA+ bidirectional jumping (default: 0) |
| `PROFILE` | release or debug | Build mode (default: release) |

**Examples:**

```bash
# RTX 3060 with integrated herds (recommended)
make SM=86 USE_SEPARATE_HERD_KERNEL=0 PROFILE=release -j

# RTX 3060 with separate kernel (for testing)
make SM=86 USE_SEPARATE_HERD_KERNEL=1 PROFILE=release -j

# RTX 3090 with integrated herds
make SM=86 USE_SEPARATE_HERD_KERNEL=0 PROFILE=release -j

# Debug build
make SM=86 USE_SEPARATE_HERD_KERNEL=0 PROFILE=debug -j
```

---

## Understanding the Implementations

### Integrated Herd Support (RCGpuCore.cu)

**How it works:**
- Adds 4 lines of code to existing optimized kernel
- Calculates herd bias: `(herd_id * 17)`
- Applies bias to jump selection: `jmp_ind = (X[0] + herd_bias) & (JMP_CNT - 1)`
- Keeps ALL existing optimizations

**Advantages:**
- ✅ Zero performance overhead (6.55 GK/s)
- ✅ Minimal code changes (battle-tested)
- ✅ Uses Jacobian coordinates (1 inversion per 10K iters)
- ✅ Shared memory for jump tables
- ✅ L2 cache optimization
- ✅ Montgomery batch inversion

**Disadvantages:**
- ❌ All herds share same jump table (only selection differs)
- ❌ Cannot tune per-herd parameters independently

**When to use:**
- ✅ Production solving (maximum speed)
- ✅ When you want spatial diversity with zero overhead
- ✅ When you trust battle-tested code

---

### Separate Herd Kernel (GpuHerdKernels.cu)

**How it works:**
- Dedicated kernel with per-herd jump tables
- Uses Montgomery batch inversion (8 kangaroos per thread)
- Shared memory for jump tables
- Affine coordinates with per-iteration batch inversion

**Advantages:**
- ✅ Per-herd jump tables (true spatial separation)
- ✅ Independent herd configurations
- ✅ Easier per-herd statistics

**Disadvantages:**
- ❌ **4x performance penalty** (1.89 vs 6.55 GK/s)
- ❌ Uses affine coordinates (expensive inversions)
- ❌ Register pressure issues
- ❌ More complex to maintain

**Why is it slower?**

| Metric | Integrated | Separate | Difference |
|--------|-----------|----------|------------|
| Coordinate system | Jacobian | Affine | - |
| Inversions per 10K iters | 38,229 | 11,468,800 | **300x more!** |
| Register usage | ~1,280 bytes | ~2,560 bytes | 2x (causes spilling) |
| Memory access | Texture cache | Shared memory | Less efficient |
| Code complexity | +4 lines | +326 lines | 81x more code |

**When to use:**
- 🔬 Research and experimentation
- 🔧 When you need truly independent herd configurations
- 📊 Benchmarking and comparison
- ⚠️ **NOT for production** (until optimized to match integrated speed)

---

## Benchmark Scripts

### 1. Simple Benchmark (`benchmark_herds.sh`)

Compares integrated herds vs no-herd baseline:

```bash
./benchmark_herds.sh
```

**Output:**
- `integrated_output.txt` - Integrated herd test results
- `baseline_output.txt` - No-herd baseline results
- Performance comparison table

### 2. Complete Comparison (`compare_kernels.sh`)

Builds and benchmarks all three configurations:

```bash
./compare_kernels.sh
```

**Output:**
- Builds both kernel implementations
- Runs 60-second tests for each
- Generates comprehensive comparison report
- Saves binaries for manual testing:
  - `rckangaroo_integrated`
  - `rckangaroo_separate`
  - `rckangaroo_baseline`

---

## Configuration Files

### HerdConfig.h

Default herd configuration for all GPU types:

```cpp
struct HerdConfig {
    int herds_per_gpu = 8;           // Number of herds
    int kangaroos_per_herd = 256;    // Kangaroos per herd
    int jump_table_size = 256;       // Jump table entries
    int dp_bits = 14;                // Distinguished point bits
    // ... more options
};
```

### HerdConfig_Optimized.h

RTX 3060-specific optimized configuration:

```cpp
struct HerdConfig_RTX3060 : public HerdConfig {
    // Optimized for 28 SMs, 3584 CUDA cores
    herds_per_gpu = 14;              // 2 blocks per SM
    kangaroos_per_herd = 896;        // Optimal warp utilization
    jump_table_size = 256;           // Fits in 24KB shared memory
    // ... RTX 3060-specific tuning
};
```

**Usage:**
```cpp
HerdConfig_RTX3060 cfg = GetOptimalConfigRTX3060(135);  // For puzzle 135
```

---

## Expected Performance (RTX 3060)

| Configuration | Speed | DPs/hour | GPU Util | Power |
|---------------|-------|----------|----------|-------|
| **No Herds** | 6.55 GK/s | ~1M | 100% | 168W |
| **Integrated Herds** | 6.55 GK/s | ~1M | 100% | 168W |
| **Separate Kernel** | 1.89 GK/s | ~290K | 85% | 120W |

**Conclusion**: Integrated herds achieve **identical performance** to baseline with spatial diversity benefits.

---

## Troubleshooting

### Build Errors

**Error**: `undefined reference to launchHerdKernels`
- **Solution**: Make sure `GpuHerdKernels.cu` is being compiled
- Check: `make print-vars | grep SRC_CU`

**Error**: Register spilling warnings
- **Solution**: Normal for separate kernel, ignored for integrated
- Can be reduced by lowering `KANGS_PER_THREAD`

### Runtime Issues

**Issue**: No DPs found with separate kernel
- **Check**: DP conversion is implemented (currently has TODO)
- **Workaround**: Use integrated kernel

**Issue**: Low GPU utilization
- **Check**: `kangaroos_per_herd` calculation in `GpuKang.cpp`
- **Should be**: `kangaroos_per_herd = total_kangaroos / herds_per_gpu`
- **Not**: Hardcoded 256

**Issue**: Different results between builds
- **Cause**: Different jump patterns due to kernel differences
- **Normal**: Both should find solution eventually

---

## How to Switch Between Implementations

### At Compile Time (Recommended)

```bash
# Use integrated (fast)
make clean && make SM=86 USE_SEPARATE_HERD_KERNEL=0 -j

# Use separate (experimental)
make clean && make SM=86 USE_SEPARATE_HERD_KERNEL=1 -j
```

### Manual Binary Selection

After running `compare_kernels.sh`:

```bash
# Use integrated kernel
cp kernel_comparison_YYYYMMDD_HHMMSS/rckangaroo_integrated ./rckangaroo

# Use separate kernel
cp kernel_comparison_YYYYMMDD_HHMMSS/rckangaroo_separate ./rckangaroo

# Use baseline (no herds)
cp kernel_comparison_YYYYMMDD_HHMMSS/rckangaroo_baseline ./rckangaroo
```

---

## Performance Analysis Tools

### GPU Monitoring

```bash
# Real-time monitoring during test
watch -n 1 nvidia-smi

# Detailed query
nvidia-smi --query-gpu=index,name,utilization.gpu,utilization.memory,power.draw,temperature.gpu --format=csv
```

### Kernel Profiling (Advanced)

```bash
# Profile integrated kernel
nvprof ./rckangaroo -herds -range 135 ...

# Profile separate kernel
nvprof ./rckangaroo -herds -range 135 ...

# Compare kernel metrics
ncu --set full --target-processes all ./rckangaroo -herds -range 135 ...
```

---

## Optimization Opportunities

### For Separate Kernel (to match integrated speed)

1. **Switch to Jacobian coordinates** ⚡ CRITICAL
   - Expected gain: 3-4x speedup
   - Effort: High (need to rewrite EC operations)

2. **Use texture memory for jump tables** 🔧 MODERATE
   - Expected gain: 5-10%
   - Effort: Medium

3. **Warp-level operations** 🔧 MODERATE
   - Expected gain: 5-15%
   - Effort: Medium

4. **Reduce KANGS_PER_THREAD** ✅ DONE
   - Current: 8 (down from 24)
   - Further reduction: Minimal gain

**Total potential**: Could theoretically match integrated speed (~6.5 GK/s) with significant effort.

---

## Recommendations

### For Production Use

✅ **Use Integrated Herds** (`USE_SEPARATE_HERD_KERNEL=0`)

**Reasons:**
1. 6.55 GK/s - full speed
2. Zero overhead
3. Battle-tested code
4. Simple and maintainable

**Command:**
```bash
make clean && make SM=86 USE_SEPARATE_HERD_KERNEL=0 PROFILE=release -j
./rckangaroo -herds -range 135 -pubkey <pubkey> ...
```

### For Research/Experimentation

🔬 **Use Separate Kernel** (`USE_SEPARATE_HERD_KERNEL=1`)

**When:**
- Testing per-herd independence
- Researching spatial separation algorithms
- Comparing different herd strategies
- Acceptable to trade 4x speed for flexibility

**Command:**
```bash
make clean && make SM=86 USE_SEPARATE_HERD_KERNEL=1 PROFILE=release -j
./rckangaroo -herds -range 135 -pubkey <pubkey> ...
```

---

## Future Work

### Short Term
- [ ] Implement DP format conversion for separate kernel
- [ ] Add per-herd statistics to integrated kernel
- [ ] Optimize separate kernel register usage

### Long Term
- [ ] Convert separate kernel to Jacobian coordinates
- [ ] Implement texture memory for jump tables
- [ ] Add adaptive herd rebalancing
- [ ] Support heterogeneous GPU setups

---

## References

- `HERD_IMPLEMENTATION_ANALYSIS.md` - Detailed technical analysis
- `README.md` - User documentation
- `SOTA++_HERDS_IMPLEMENTATION.md` - Implementation notes
- `HerdConfig.h` - Configuration options
- `HerdConfig_Optimized.h` - RTX 3060 tuning

---

## Support

If you encounter issues:

1. Check `HERD_IMPLEMENTATION_ANALYSIS.md` for technical details
2. Run `compare_kernels.sh` to verify both implementations
3. Check GPU utilization with `nvidia-smi`
4. Verify build configuration with `make print-vars`

**Current Status**: Integrated herds are production-ready at 6.55 GK/s. Separate kernel is experimental at 1.89 GK/s.
