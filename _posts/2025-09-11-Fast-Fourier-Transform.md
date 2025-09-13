---
layout: post
title: "Fast Fourier Transform (FFT): A Gentle Introduction"
author: N Dev
usemathjax: true
---

# Fast Fourier Transform (FFT): A Gentle Introduction

## 1. Introduction  
The Fourier transform, a pivotal operation in signal processing and linear systems analysis, maps a function from its native domain (e.g., time or space) to the frequency domain. This transformation is invaluable due to a fundamental property known as the Convolution Theorem, which states that the convolution of two functions in the time domain corresponds to the pointwise multiplication of their respective Fourier transforms in the frequency domain. 

## 2. The Discrete Fourier Transform (DFT)  
The DFT transforms a finite, discrete sequence of data points into a corresponding finite, discrete sequence of frequency components. This output sequence represents samples of the continuous Discrete-Time Fourier Transform (DTFT) of the original signal.

The DFT maps a sequence of complex numbers $$x[n]$$ of length $$N$$ into another sequence $$X[k]$$ : 


$$
X[k] = \sum_{n=0}^{N-1} x[n] \, e^{-i 2\pi kn / N}, 
\quad k = 0,1,\ldots,N-1
$$  

where $$i = \sqrt{-1}$$.

The FFT is fast $$O(nlogn)$$ algorithm  to compute the DFT which by itself is an $$O(n^2)$$ algorithm

###  C++ implementation of DFT
To understand how DFT works we implement it in C++ as below. Note our arrays $$X_n$$ and $$X_k$$ are both of length $$2N$$ where $$N$$ is the lowest power of 2 greater than the number of samples. Each element of $$X_n$$ and $$X_k$$ is a complex number that is stored as two consecutive floats: the real part followed by the imaginary part.

If $$x[n] = a + ib$$, then

$$
X[k] = \sum_{n=0}^{N-1} (a + ib)(\cos \theta + i \sin \theta), 
\quad \theta = -\tfrac{2\pi kn}{N}
$$

So both real and imaginary parts of the input contribute:

$$
\Re(X[k]) = \sum_{n=0}^{N-1} \big(a \cos \theta - b \sin \theta\big)
$$ 

and 

$$
\Im(X[k]) = \sum_{n=0}^{N-1} \big(a \sin \theta + b \cos \theta\big)
$$ 

```

void discreteFourierTransform(const float *X_n, float *X_k, int N) {
    constexpr float PI = 3.14159265358979323846f;

    for (int k = 0; k < N; k++) {
        float real = 0.0f, imag = 0.0f;

        float angle_step = -2.0f * PI * k / N;
        float cos_step   = std::cos(angle_step);
        float sin_step   = std::sin(angle_step);

        float cos_t = 1.0f, sin_t = 0.0f;

        for (int n = 0; n < N; n++) {
            float a = X_n[2*n];     
            float b = X_n[2*n+1];   

            real += a * cos_t - b * sin_t;
            imag += a * sin_t + b * cos_t;

            float tmp = cos_t * cos_step - sin_t * sin_step;
            sin_t     = cos_t * sin_step + sin_t * cos_step;
            cos_t     = tmp;
        }

        X_k[2*k]   = real;
        X_k[2*k+1] = imag;
    }
}
```

## 3. The DFT Matrix
The DFT process can be expressed as a matrix operation where the original vector $$X_n$$ is transformed into the vector $$X_k$$ via multiplication with the square DFT Matrix, $$W$$ where $$W$$ is ad defined below 


$$
W = \left( \omega^{jk} \right)_{j,k=0,\ldots,N-1}
$$

or equivalently:

$$ 
W = 
\begin{bmatrix}
1 & 1 & 1 & 1 & \cdots & 1 \\
1 & \omega & \omega^{2} & \omega^{3} & \cdots & \omega^{N-1} \\
1 & \omega^{2} & \omega^{4} & \omega^{6} & \cdots & \omega^{2(N-1)} \\
1 & \omega^{3} & \omega^{6} & \omega^{9} & \cdots & \omega^{3(N-1)} \\
\vdots & \vdots & \vdots & \vdots & \ddots & \vdots \\
1 & \omega^{N-1} & \omega^{2(N-1)} & \omega^{3(N-1)} & \cdots & \omega^{(N-1)(N-1)}
\end{bmatrix}
$$ 

where $$\omega = e^{-2 \pi i / N}$$ is the primitive $$N$$-th root of unity with $$i^2 = -1$$.  

Note that  

$$
\omega^x = \omega^{x \bmod N}.
$$

## 4. From DFT to FFT via DFT Matrix
The above matrix $$W$$ has a lot of symmetry
- The entries are just powers of a single number $$\omega$$.
- Many values repeat: for example, $$\omega^{j+N} = \omega^j$$.
- Rows and columns are related by periodicity.

In our formulation $$N$$ is a power of 2 i.e. it is even. This implies that any index $$n$$ can be written as either even $$2m$$ or odd $$2m+1$$.
We split the input vector as  

$$
\begin{bmatrix}
x_{\text{even}} \\
x_{\text{odd}}
\end{bmatrix}
$$  

where  

$$
x_{\text{even}} = (x[0], x[2], x[4], \dots)
$$

$$
x_{\text{odd}} = (x[1], x[3], x[5], \dots)
$$

Then the DFT can be written as  

$$
X[k] =\ \sum_{m=0}^{\tfrac{N}{2}-1} x[2m]  \omega^{2mk} 
+\ \omega^k \sum_{m=0}^{\tfrac{N}{2}-1} x[2m+1]  \omega^{2mk}.
$$  

Notice that both sums are themselves **DFTs of length \(N/2\)** (with base root $$(\omega^2)$$.

The DFT matrix can be decomposed, recursively,  as  

$$
W_N =
\begin{bmatrix}
W_{N/2} & D_{N/2} W_{N/2} \\
W_{N/2} & -D_{N/2} W_{N/2}
\end{bmatrix}
$$

where:

- $$W_{N/2}$$ is the DFT matrix of size $$(N/2)$$,  
- $$D_{N/2}$$ is a diagonal matrix of *twiddle factors*:  
  $$D_{N/2} = \mathrm{diag}(1, \omega, \omega^2, \ldots, \omega^{N/2 - 1}).$$

Since $$n$$ is a power of two there are $$log(n)$$ steps in the reduction. If the multiplications in each step are of $$O(n)$$ then we have an $$O(nlogn)$$ algorithm for FFT.

Both terms are size- $$(N/2)$$ DFTs (with root $$\omega^2$$ . Call them $$E[k]$$ and $$O[k]$$. 

- For $$(0 \leq k < N/2)$$:  
  $$X[k] = E[k] + \omega^k O[k]$$

- For $$(N/2 \leq k < N)$$:  
  $$X[k] = E[k - N/2] - \omega^{k - N/2} O[k - N/2]$$

  Thus except for the multiplcative part the odd and even terms in the second equation above are already computed. 

This is the **butterfly relation**.

### Why FFT works
The key insight behind the FFT is reusing computation. In a naive DFT, every output $$X[k]$$ sums over all $$N$$ inputs independently, leading to 
$$O(N^2)$$ operations. 

Each sum is itself a smaller DFT of length $$N/2$$. By computing these smaller DFTs once and reusing them, we avoid repeating calculations. Recursively applying this divide-and-conquer approach reduces the complexity from $$O(N^2)$$ to $$O(Nlog⁡N)$$

## 5. The recursive algorithm for FFT
The recursive divide and conquer algorithm is then easier to understand from the above formulation and its pseudocode is given as 
```
def fft_recursive(x,n,s) :
    if n == 1 :
        X_0 = X_0
    else :
        X_0,...,X_n/2-1 = fft_recursive(x,n/2,s)
        X_n/2,...,X_n   = fft_recursive(x+s,n/2,s)
        for k in range(0,n/2-1) :
                p = X_k
                q = omega * X_k+N/2
              X_k = p + q
          X_k+N/2 = p - q
    return X_1,...,X_n   
```
## 6. The Cooley-Tukey Iterative FFT Algorithm

The Cooley-Tukey Iterative FFT is one of the most common FFT algorithms, the pseudocode for which is given below.
```
def fft_iterative(x,n) :
    bit_reverse(x,X)
    n = len(x)
    m = 2
    for s in range(1,log(n)) :
        w_m = omega
        for k in range(0,n-1,m) :
            w = 1
            for j in range(0,m/2) :
                t = wX[k + j + m/2]
                u =  X[k + j]
                X[k + j] = u + t
                X[k + j + m/2] = u - t
                w = w * w_m
        m *= 2
    return X
```
The $$bit-reverse$$ algorithm reorders input indices by reversing their binary representations, preparing the data so that the recursive even–odd splits of the FFT can be executed efficiently in place.

So for N = 8 the situation is as below

$$
\begin{bmatrix}
Index & Binary & Bit reversed & New Index \\
0 & 000 & 000 & 0 \\
1 & 001 & 100 & 4 \\
2 & 010 & 010 & 2 \\
3 & 011 & 110 & 6 \\
4 & 100 & 001 & 1 \\
5 & 101 & 101 & 5 \\
6 & 110 & 011 & 3 \\
7 & 111 & 111 & 7 \\
\end{bmatrix}
$$

| Index | Binary | Bit-reversed | New index |
| ------| ------ | ------------ | --------- |
| 0     | 000    | 000          | 0         |
| 1     | 001    | 100          | 4         |
| 2     | 010    | 010          | 2         |
| 3     | 011    | 110          | 6         |
| 4     | 100    | 001          | 1         |
| 5     | 101    | 101          | 5         |
| 6     | 110    | 011          | 3         |
| 7     | 111    | 111          | 7         |

## 7. The C++ implementation of iterative C++

## 8. The inverse FFT

## 9. References

- [Discrete Fourier Transform — Wikipedia](https://en.wikipedia.org/wiki/Discrete_Fourier_transform)  
- [Fast Fourier Transform — Wikipedia](https://en.wikipedia.org/wiki/Fast_Fourier_transform)  
- [Cooley–Tukey FFT Algorithm — Wikipedia](https://en.wikipedia.org/wiki/Cooley%E2%80%93Tukey_FFT_algorithm)
- [DFT Matrix - Wikipedia](https://en.wikipedia.org/wiki/DFT_matrix)






