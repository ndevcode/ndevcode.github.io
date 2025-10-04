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
