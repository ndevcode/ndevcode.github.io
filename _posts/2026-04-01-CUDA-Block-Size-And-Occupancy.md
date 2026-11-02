---
layout: post
title: "Why a Bigger CUDA Block Can Be Slower, Then Faster Again"
author: N Dev
usemathjax: true
---

# Why a Bigger CUDA Block Can Be Slower, Then Faster Again

## Introduction

I implemented a 3D convolution kernel where each block loads a tile of the
input into shared memory, one depth slice at a time. Each thread then computes
one output element from the tile.

Initially I used a $$32 \times 32$$ thread block. My assumption was that a
larger block should allow more reuse from shared memory. However changing the
block size to $$16 \times 16$$ made the kernel faster while producing the same
output.

So my first conclusion was that smaller blocks are faster. This turned out to
be incorrect.

## The Kernel Launch

The block size is a launch parameter as below.

```
dim3 block(32, 32);
dim3 grid((output_cols + block.x - 1) / block.x,
          (output_rows + block.y - 1) / block.y,
          output_depth);

conv3d_kernel<<<grid, block, shmem_size>>>(input, output, /* dimensions */);
```

I tested the kernel on an RTX 4060 Laptop GPU using FP32. The input size was
$$256 \times 256 \times 256$$ and the convolution kernel size was
$$5 \times 5 \times 5$$. The code was compiled with `-O3`.

Each result below is the median time from 30 kernel runs measured using CUDA
events.

## Performance for Different Block Sizes

| Block | Threads | Warps | Blocks per SM | Occupancy | Time |
|---|---|---|---|---|---|
| 8×8 | 64 | 2 | 24 | 99% | 4.8 ms |
| 16×16 | 256 | 8 | 6 | 99% | 4.6 ms |
| 32×16 | 512 | 16 | 3 | 99% | 4.8 ms |
| 32×24 | 768 | 24 | 2 | 99% | 4.9 ms |
| 32×25 | 800 | 25 | 1 | 52% | 7.4 ms |
| 32×26 | 832 | 26 | 1 | 54% | 6.8 ms |
| 32×28 | 896 | 28 | 1 | 58% | 6.4 ms |
| 32×32 | 1024 | 32 | 1 | 66% | 6.0 ms |

From $$8 \times 8$$ to $$32 \times 24$$ the execution time remains almost the
same. So a smaller block by itself does not make the kernel faster.

However increasing the block size from 768 threads to 800 threads makes the
kernel around $$1.5$$ times slower. After this point increasing the number of
threads again improves performance. The $$32 \times 32$$ block is faster than
the smaller $$32 \times 25$$ block.

The shape of the block is not the important part here. I also tested a
$$64 \times 16$$ block. It has 1024 threads and takes around 5.9 ms, which is
almost the same as the $$32 \times 32$$ block.

## NSight Compute Occupancy Results

I used the Occupancy section in `ncu` to understand this behaviour.

```
ncu --section Occupancy --kernel-name conv3d_kernel ./a.out
```

For the $$32 \times 32$$ block NSight Compute reports

```
Block Limit Warps                     block            1
Theoretical Active Warps per SM        warp           32
Theoretical Occupancy                     %        66.67
Achieved Occupancy                        %        66.42
```

For the $$16 \times 16$$ block it reports

```
Block Limit Warps                     block            6
Theoretical Active Warps per SM        warp           48
Theoretical Occupancy                     %          100
Achieved Occupancy                        %        98.83
```

An SM on this GPU has 48 warp slots. Also all the warps belonging to a block
must be placed on the same SM. Therefore the active number of warps depends on
how many complete blocks can fit on the SM.

The situation is as below.

```
24 warps per block : 2 blocks fit -> 48 warps running
25 warps per block : 1 block fits -> 25 warps running
32 warps per block : 1 block fits -> 32 warps running
```

For a block containing 24 warps, two complete blocks can reside on the SM and
all 48 warp slots are used. Adding one more warp to the block means that two
blocks no longer fit. The active warp count then drops from 48 to 25.

Once only one block fits, every additional warp in that block increases the
number of active warps. So the active warp count increases from 25 for the
$$32 \times 25$$ block to 32 for the $$32 \times 32$$ block. This explains why
the larger block becomes faster again.

## Verifying the Explanation

Before measuring all the intermediate sizes I expected the $$32 \times 24$$
block to be fast and the $$32 \times 25$$ block to be slower than the
$$32 \times 32$$ block. If the number of available warp slots was the reason,
the performance should change at exactly this boundary. The measurements show
the expected behaviour.

Occupancy does not explain the complete performance of the kernel. Increasing
the active warp count from 25 to 48 is nearly a factor of two, while the kernel
only becomes around $$1.5$$ times faster. This implies that another resource
starts limiting the performance. I have not isolated that resource yet.

Note that `ncu` also reports register and shared memory limits of one block for
the $$32 \times 32$$ configuration. So the number of warp slots is not the only
resource limit for this block size.

## Conclusion

The block size is not a smooth performance tradeoff. There are boundaries at
which one fewer complete block fits on the SM and occupancy drops suddenly.
Also once full occupancy is reached, making the block smaller may provide no
improvement, as can be seen for the $$8 \times 8$$ and $$16 \times 16$$ block
sizes.

The number of available warp slots and other resource limits differ between
GPUs. Therefore the boundary will not necessarily occur at the same block size
on another GPU. The Occupancy section in `ncu` is useful to check this before
tuning the block dimensions.

I also verified the output against a double precision CPU implementation for
each block size using a smaller input. The maximum error was around `2e-5`,
which is expected from FP32 rounding. For the $$16 \times 16$$ and
$$32 \times 32$$ blocks I also checked the complete
$$256 \times 256 \times 256$$ input. The maximum error in that case was around
`4e-5`.
