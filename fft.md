\* Note: To render LaTeX math equations, view the doc in [VSCode](https://code.visualstudio.com/) + [Markdown All in One](https://github.com/yzhang-gh/vscode-markdown) plugin

$$
\global\def\cancelto#1#2{\bcancel{#2} \underset{\mathrlap{\hphantom{.} #1}}{}} %%\cancelto{null}{1}
$$

# Fourier transform

Discrete Fourier transform:

$$
X(k)=\sum_{n=0}^{N-1}x(n)e^{{-j2 \pi nk}\over{N}} \xrightarrow{\tiny{exp(-j2\pi/N)=W_N}}\sum_{n=0}^{N-1}x(n)W^{nk}_N
$$

***

## 2-pass decomposition

- With $N=N_1N_2$
  - Fold $n$ into $(n_1,n_2)$ where $n=n_1+N_1n_2$
  - Fold $k$ into $(k_2,k_1)$ where $k=k_2+N_2 k_1$ (dimension reversed/transposed)
- Fourier transform $X(k)=\Sigma_{n=0}^{N-1}x(n)W^{nk}_N$ becomes

$$
\begin{aligned}
\underbrace{X(k_2+N_2 k_1)}_{X(k_2, k_1)} &= \sum_{n_1}\sum_{n_2}x(n_1 + N_1 n_2)W_{N_1 N_2}^{(n_1+N_1n_2)(k_2+N_2k_1)} \\
&= \sum_{n_1}\sum_{n_2}x(\dotsb)W^{n_1 k_1}_{N_1} W^{n_2 k_2}_{N_2} W^{n_1 k_2}_{N_1 N_2} \cancelto{e^{j 2 \pi N}=1}{W^{n_2 k_1}_{1}} \\
&= \underbrace{\sum_{n_1} \underbrace{\vphantom{\sum_{n_2}}W^{n_1 k_2}_{N_1 N_2}}_{\text{Twiddle}}
  \biggl(
    \underbrace{
      \sum_{n_2}x(\dotsb)W^{n_2 k_2
    }_{N_2}}_{x'(n_1,k_2)}
  \biggr)
W^{n_1 k_1}_{N_1}}_{x''(k_1,k_2)}
\end{aligned}
$$

- Algorithm
  1. Perform FFTs along the high dimension $\mathbf{D_2}$
  2. Pointwise multiply by twiddle factor $W^{n_1 k_2}_{N_1 N_2}$
  3. Perform FFTs along the low dimension $\mathbf{D_1}$
  4. Read the resulting data transposed

***

## 3-pass decomposition

- With $N=N_1N_2N_3$
  - Fold $n$ into $(n_1,n_2,n_3)$ where $n=n_1+N_1n_2+N_1N_2n_3$
  - Fold $k$ into $(k_3,k_2,k_1)$ where $k=k_3+N_3k_2+N_2 N_3k_1$ (dimension reversed/transposed)
- Fourier transform becomes:

$$
\begin{aligned}
\underbrace{X(k_3+N_3k_2+N_2 N_3k_1)}_{X(k_3, k_2, k_1)} &= \sum_{n_1}\sum_{n_2}\sum_{n_3}x(n_1+N_1n_2+N_1N_2n_3)W_{N_1N_2N_3}^{(n_1+N_1n_2+N_1N_2n_3)(k_3+N_3k_2+N_2 N_3k_1)} \\
&= \sum_{n_1}\sum_{n_2}\sum_{n_3}x(\dotsb)W^{n_1 k_1}_{N_1} W^{n_2 k_2}_{N_2} W^{n_3 k_3}_{N_3} W^{n_2 k_3}_{N_2 N_3} \cancelto{W^{n_1(N_3 k_2 + k_3)}_{N_1 (N_2 N_3)}}{W^{n_1 k_2}_{N_1 N_2} W^{n_1 k_3}_{N_1 N_2 N_3}} \\
%%&= \sum_{n_1} W^{n_1 k_2}_{N_1 N_2} W^{n_1 k_3}_{N_1 N_2 N_3} \biggl[ \sum_{n_2} W^{n_2 k_3}_{N_2 N_3} \biggl( \sum_{n_3}x(\dotsb)W^{n_3 k_3}_{N_3} \biggr) W^{n_2 k_2}_{N_2} \biggr] W^{n_1 k_1}_{N_1} \\
&= \underbrace{
\sum_{n_1} \underbrace{ \vphantom{\sum_{n_1}} W^{n_1(N_3 k_2 + k_3)}_{N_1 (N_3 N_2)} }_{\text{Twiddle(!)}}
\biggl[
\underbrace{
  \sum_{n_2} \underbrace{
    \vphantom{\sum_{n_1}} W^{n_2 k_3}_{N_2 N_3} }_{\text{Twiddle} \vphantom{\Text{()}}}
  \biggl(
  \underbrace{
    \sum_{n_3} x(\dotsb)W^{n_3 k_3}_{N_3} }_{x'(n_1, n_2, k_3)} \vphantom{\Text{()}}
  \biggr)
W^{n_2 k_2}_{N_2} }_{x''(n_1, k_2, k_3 )}
\biggr]
W^{n_1 k_1}_{N_1}
}_{x'''(k_1, k_2, k_3)}\\
\end{aligned}
$$
> (!) Note: outermost twiddle factor has it's index **transposed/bit-reversed**

- Recursive tree structure revealed:
  - FFT for $N_1 N_2 N_3$ array requires 2-pass FFT for $N_1$-by-$N_3 N_2$ pseudo matrix
  - FFT for $N_3 N_2$ array requires *another 2-pass FFT* for $N_2$-by-$N_3$ pseudo matrix
  - 3 leaf nodes corresponding to FFTs of 3 different radices, 2 parent nodes corresponding to 2 twiddle multiplication
  - Recursion also implies FFT begins with highest pseudo-dim. Never begins with lowest pseudo-dim.

```txt
  N1xN2xN3 .______ N1
            \_____ N2xN3 .____ N2
                          \___ N3
```

***

## Tree decomposition
- Example input: 256-point signal
- Decompose into 4-pass radix-4 FFTs
   - Perform radix-4 FFTs along dim $\mathbf{D_4}$; pointwise-multiply twiddle factor $W_{N_3 N_4}$
   - Perform radix-4 FFTs along dim $\mathbf{D_3}$; pointwise-multiply twiddle factor $W_{N_2 (N_4 N_3)^{(!)}}$
   - Perform radix-4 FFTs along dim $\mathbf{D_2}$; pointwise-multiply twiddle factor $W_{N_1 (N_4 N_3 N_2)^{(!)}}$
   - Perform radix-4 FFTs along dim $\mathbf{D_1}$
   - Read the data transposed
```txt
  256 (N1xN2xN3xN4) .____ 4  (N1)
                     \___ 64 (N2xN3xN4) .____ 4  (N2)
                                         \___ 16 (N3xN4) .____ 4 (N3)
                                                          \___ 4 (N4)
```
<!-- - For $i$ in $1:4$:
  - Perform radix-4 FFT along ${N_{4-i+1}}$; pointwise-multiply twiddle factor $W_{\prod_{j=4-i+1}^{4}{N_j}} \text{ if } i>1$ -->
<!-- Question
- How to make tree traversal deepest child node first? -->

- Alternative Method - Decompose into pseudo-2D 16x16 matrix
   - Perform 2-pass radix-4 FFTs along $\mathbf{D_{N_3 N_4}}$ pseudo-dimension; pointwise-multiply twiddle factor $W_{N_3 N_4}$
   - Pointwise-multiply twiddle factor $W_{(N_1 N_2)(N_4 N_3)^{(!)}}$
   - Perform 2-pass radix-4 FFTs along $\mathbf{D_{N_1 N_2}}$ pseudo-dimension; pointwise-multiply twiddle factor $W_{N_1 N_2}$
   - Read the data transposed
```txt
  256 (N1xN2xN3xN4) .____ 16 (N1xN2) .____ 4 (N1)
                    |                 \___ 4 (N2)
                    |
                     \___ 16 (N3xN4) .____ 4 (N3)
                                      \___ 4 (N4)
```
> (!) Note: again, indicates that twiddle factor has it's index **transposed/bit-reversed**

***

## Progressive transposition (Stockham algorithm)

- Instead of transposing all dimensions at once in the end, transpose one at a time as we do the butterfly
- Access pattern is more sequential--good for the cache (see: Geometric View of Stockham Algorithm)
<!-- - TODO: Example
  - Before: FFT pass 0 -> LDS exchange -> FFT pass 1 -> WB+T
  - Now:    FFT -> WB -> FFT -> WB+T -->

```txt
  256 (N1xN2xN3xN4) .____ 16 (N1xN2) .____ 4 (N1) ____* 16 (N2xN1) ____* 256 (N4xN3xN2xN1)
                    |                 \___ 4 (N2) ___/                 |
                    |                                                  |
                     \___ 16 (N3xN4) .____ 4 (N3) ____* 16 (N4xN3) ___/
                                      \___ 4 (N4) ___/
```

***


## Decimation-in-freqeuncy (DIF) form:
- DIF form is derived by factoring twiddle factor the different way:

$$
\begin{aligned}
X(\dotsb) &= \sum_{n_1}\sum_{n_2}\sum_{n_3}x(\dotsb)W^{n_1 k_1}_{N_1} W^{n_2 k_2}_{N_2} W^{n_3 k_3}_{N_3} W^{n_2 k_3}_{N_2 N_3} \cancelto{W^{n_1(N_3 k_2 + k_3)}_{N_1 (N_2 N_3)}}{W^{n_1 k_2}_{N_1 N_2} W^{n_1 k_3}_{N_1 N_2 N_3}}  \\
&= \sum_{n_1} \underbrace{ \vphantom{\sum_{n_1}} W^{n_1(N_3 k_2 + k_3)}_{N_1 (N_3 N_2)} }_{\text{Twiddle}}
\biggl[
  \sum_{n_2} \underbrace{
    \vphantom{\sum_{n_1}} W^{n_2 k_3}_{N_2 N_3} }_{\text{Twiddle} \vphantom{\Text{()}}}
  \biggl(
    \sum_{n_3} x(\dotsb)W^{n_3 k_3}_{N_3}
  \biggr)
W^{n_2 k_2}_{N_2}
\biggr]
W^{n_1 k_1}_{N_1}
\hphantom{.....}\text{(decimation in time)}\\
or & \\
& = \sum_{n_1}\sum_{n_2}\sum_{n_3}x(\dotsb)W^{n_1 k_1}_{N_1} W^{n_2 k_2}_{N_2} W^{n_3 k_3}_{N_3} \bcancel{W^{n_2 k_3}_{N_2 N_3}} W^{n_1 k_2}_{N_1 N_2} \cancelto{W^{(n_1+N_1 n_2)k_3}_{(N_1 N_2) N_3}}{W^{n_1 k_3}_{N_1 N_2 N_3}} \\
&=
\sum_{n_1} \underbrace{ \vphantom{\sum_{n_1}} W^{n_1 k_2}_{N_1 N_2} }_{\text{Twiddle}}
\biggl[
  \sum_{n_2} \underbrace{
    \vphantom{\sum_{n_1}} W^{(n_1+N_1 n_2)k_3}_{(N_1 N_2) N_3} }_{\text{Twiddle} \vphantom{\Text{()}}}
  \biggl(
    \sum_{n_3} x(\dotsb)W^{n_3 k_3}_{N_3}
  \biggr)
W^{n_2 k_2}_{N_2}
\biggr]
W^{n_1 k_1}_{N_1} \hphantom{.....}\text{(decimation in frequency)}\\
\end{aligned}
$$
- Comparing the equations: all except twiddle factors stay the same
  - DIT: coarser twiddle $W_{N_2 N_3}$ $\rightarrow$ finer twiddle $W_{N_1(N_3 N_2)}$
  - DIF: finer twiddle $W_{(N_1 N_2) N_3}$ $\rightarrow$ coarser twiddle $W_{N_1 N_2}$

***

## Complex conjugation property
- Complex conjugation property: $x^*(n)\xrightarrow{\text{DFT}_N}X^*(N-k)$. Proof:

$$
\begin{aligned}
\text{DFT}_N\bigl\{x^*(n)\bigr\} & = \sum_{n}^{N}{x^*(n)W_{N}^{nk}}
= \biggl[\sum_{n}^{N}{x(n)W_{N}^{-nk}}\biggr]^*
= \biggl[\sum_{n}^{N}{
  x(n)  W_{N}^{-nk}\cancelto{=1}{ W_N^{nN}} \hphantom{....}
  }\biggr]^* \\
&= \biggl[\sum_{n}^{N}{
  x(n)  W_{N}^{n(N-k)}
  }\biggr]^*
= X^*(N-k)
\end{aligned}
$$

- Conjugate symmetry For R2C FFT: $X(k)=X^*(N-k)$. Proof:


$$
\begin{aligned}
\text{FFT}\bigl\{x(n)\bigr\} &= \text{FFT}\bigl\{x^*(n)\bigr\} \\
X(k) &= X^*(N-k)
\end{aligned}
$$

***

## Real-valued FFT algorithms

### Forward R2C
1. Make up complex valued sequence $z(n)$ from real-valued sequence $x(n)$
  $$ z(n) = \underbrace{x(2n)}_{x_e(n)} + j\ \underbrace{x(2n+1)}_{x_o(n)} , \qquad n=0, ..., {N \over 2}-1 $$
2. Transform $z(n)$ into $Z(k)$. From which extract real/imag parts $X_e(k)$ and $X_o(k)$
  $$\begin{aligned}
  Z(k) &= X_e(k) + j\ X_o(k) \\
       &= \underbrace{\frac{1}{2 } \biggl[Z(k) + Z^*\biggl(\frac{N}{2}-k\biggr)\biggr]}_{X_e(k)}
          + j\ \underbrace{\frac{1}{2j} \biggl[Z(k) - Z^*\biggl(\frac{N}{2}-k\biggr)\biggr]}_{X_o(k)}
  \end{aligned}$$
3. Use DIT property and complex conjugate property to reconstruct $X(k) = X_e(k) +  W^k_N X_o(k)$
  $$
  \begin{aligned}
  X(k) & = X_e(k) + W^k_N X_o(k) &\quad \text{for}\ k=0,..., {N \over 2} - 1 \\
  X(k) & = X_e(k) + W^0_N X_o(k) = X_e(k) + X_o(k) &\quad \text{for}\ k= {N \over 2} \\
  X(k) & = X^*(N-k) &\quad \text{for}\ k={N \over 2}+1, ... N-1
  \end{aligned}
  $$

### Backward C2R
1. Obtain $X_e(k)/X_o(k)$ of the time domain sequence such that $z(n) = x_e(n) + jx_o(n)$
2. Inverse transform $Z(k) = X_e(n) + jX_o(n)$
3. $x_e = Re\{z\}, \ x_o = Im\{z\}$

***

## Appendix
### Decimation in time (DIT)
$$
\begin{aligned}
X(k) &= \sum_{n=0}^{N-1} x(n) W_N^{nk} \\
&= \sum_{n=0}^{{N/2}-1} x(2n)W_N^{2nk} + \hphantom{W^k_N} \sum_{n=0}^{{N/2}-1} x(2n+1)W_N^{2(n+1)k} \\
&= \underbrace{\sum_{n=0}^{{N/2}-1} x(2n)W_{N/2}^{nk} }_{ X_e(k)} + W^k_N \underbrace{\sum_{n=0}^{{N/2}-1} x(2n+1)W_{N/2}^{nk} }_{X_o(k)}
\tag{1}
\end{aligned}
$$

### Extract real and imag part
$$
\begin{aligned}
z(n) = x_r(n) + jx_i(n)
\end{aligned}
$$
$$
\begin{aligned}
X_r(k) &= \mathcal{F}\biggl\{ \frac{1}{2} \left( z + z^* \right) \biggr\} = \frac{1}{2}  \biggl[Z(k) + Z^*(N-k)\biggr], \qquad &k=0,...,N-1 \\
X_i(k) &= \mathcal{F}\biggl\{ \frac{1}{2j}\left( z - z^* \right) \biggr\} = \frac{1}{2j} \biggl[Z(k) - Z^*(N-k)\biggr], \qquad &k=0,...,N-1
\tag{2}
\end{aligned}
$$
## Computational complexity

A|B|C
-|-|-
d|a|a|
z|z|z|
