---
layout: post
title: "Improving Performance of Blelloch Scan Multi-Block Operation on GPU"
author: N Dev
usemathjax: true
---

# Improving Performance of Blelloch Scan Multi-Block Operation on GPU

## Introduction

Blelloch scan, also known as parallel prefix sum, is a fundamental parallel algorithm used to compute prefix sums of an input array. It has wide applications in parallel algorithms, including stream compaction, sorting, and graph processing. Efficient implementation on GPUs can significantly speed up these operations in high-performance computing and data-parallel tasks.

## Code Snippet for Both Kernels

### Blelloch Scan Shared Memory Kernel

```
__global__ void blelloch_shared_memory(float *arr, int n, float *res) {
    extern __shared__ float shmem[];

    int tid = threadIdx.x;
    int gid = blockDim.x * blockIdx.x + threadIdx.x;
    if (gid >= n) return;

    shmem[tid] = arr[gid];
    __syncthreads();

    //upsweep
    for (int d = 1; d < blockDim.x; d <<= 1) {
        int idx = (tid + 1) * d * 2 - 1;
        if (idx < blockDim.x) {
            shmem[idx]  +=  shmem[idx-d];
        }
        __syncthreads();
    }

    //copy over last element to result array
    if (tid == 0) {
        res[blockIdx.x] = shmem[blockDim.x-1];
        shmem[blockDim.x-1] = 0;
    }
    __syncthreads();

    //downsweep
    for (int d = blockDim.x/2; d > 0; d >>= 1) {
        int idx = (tid + 1) * d * 2 - 1;
        if (idx < blockDim.x) {
            float tmp = shmem[idx];
            shmem[idx] += shmem[idx-d];
            shmem[idx-d] = tmp;
        }
        __syncthreads();
    }

    arr[gid] = shmem[tid];
    __syncthreads();
}

```

### Blelloch Scan Propagate Prefix Kernel
```
__global__ void blelloch_scan_propagate_prefix(float *child, int n, const float *parent) {

    int gid = blockDim.x * blockIdx.x + threadIdx.x;
    if (gid >= n) return;

    child[gid] += parent[blockIdx.x];
}

```

## Measuring Performance for Blelloch Scan

The primary performance metric used here is **GEPS (Giga-Elements Per Second)**, representing how many billions of elements the kernel processes per second.

GEPS is chosen because it directly measures data throughput, which is intuitive for scan operations processing large element arrays where floating-point operations might not fully describe kernel workload.

### How GEPS is Computed

Given:

- Number of elements processed: \(N\)
- Kernel runtime in seconds: \(T\)

GEPS is calculated as:

\[
\text{GEPS} = \frac{N}{T \times 10^9}
\]

For example, processing \(16,777,223\) elements in \(5.6\) ms results in roughly \(3\) GEPS.

## Analysis of NSight Compute Kernel Profiling Data

| Kernel                     | SM Efficiency (%) | DRAM Utilization (%) | Shared Memory Utilization (%) | FP32 Utilization (%) | Bound Type      |
|----------------------------|-------------------|---------------------|-------------------------------|---------------------|-----------------|
| blelloch_shared_memory      | 90–99             | 10–25               | 55–66                         | ~8                  | Latency Bound   |
| blelloch_scan_propagate_prefix | 69–99          | 44–94               | ~2                            | ~4                  | Memory Bound    |

- **Latency Bound:** The shared memory kernel heavily uses shared memory and synchronizations causing latency stalls.
- **Memory Bound:** The prefix propagation kernel is constrained primarily by global memory bandwidth.

## What Can Be Done to Improve GEPS for Kernels

### Idea 1: Reduce Shared Memory Bank Conflicts

By padding shared memory array accesses to prevent banks conflicts, we reduce serialization and increase shared memory efficiency.

#### Code Snippet

