---
layout: default
---

Nilabha Dev.
I build chip design tools (20+ years in design-for-test and physical design), ship reinforcement learning in them, and write GPU kernels by hand. Notes on algorithms meeting real hardware: derivations, measurements, and what broke.

[GitHub](https://github.com/ndevcode) · [LeetGPU](https://leetgpu.com/ndevcode)

## PART 1 : High Performance Computing

- [Why a Bigger CUDA Block Can Be Slower, Then Faster Again](/2026/04/01/CUDA-Block-Size-And-Occupancy.html)  
  An occupancy cliff at 25 warps per block, explained with Nsight Compute.
- [Improving Performance of Blelloch Scan Multi-Block Operation on GPU](/2025/10/04/Improving-Blelloch-Scan.html)  
  Profiling two kernels: one latency bound, the other memory bound.

## PART 2 : C++ and Python

- [Understanding the Direction of std::is_convertible in C++](/2024/11/28/is-convertible-direction.html)  
  A backwards check, and why my tests couldn't catch it.
- [How to print a container in C++?](/2023/06/29/c++-print-array-post.html)  
  Six ways, ending with C++20 concepts.
- [Understanding Generator Expressions vs List Comprehensions in Python](/2025/04/23/understanding-generators.html)  
  A cleaner way to build a vocabulary, and what lazy evaluation gives you.

## PART 3 : Algorithms

- [Fast Fourier Transform (FFT): A Gentle Introduction](/2025/09/11/Fast-Fourier-Transform.html)  
  From the DFT matrix to an in-place C++ implementation.
