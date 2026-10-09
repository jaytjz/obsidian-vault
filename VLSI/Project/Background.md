## Introduction

**Goal:** ML accelerator beyond a general purpose MatMul unit.

**Idea progression:**  
Transformer weights embedded in silicon (cf. Taalas) → dedicated chess/Go inference chip → sparse matrix multiplication accelerator

**Driver:** area constraints narrowed the scope at each step.

## Background for Sparse Matrix Multiplication

### Target kernels 
- **Notation:** Sp = sparse, GE = general, M = matrix, V = vector; uppercase = matrix (A, B), lowercase = vector (b, c). E.g. SpMSpV = sparse matrix × sparse vector.
- **MV:** SpMV, SpMSpV (A × b = c); SpGEMV (A × b + d = c)
- **MM:** SpMM, SpMSpM (A × B = C); SpGEMM (A × B + D = C)

### Sparse Formats

- **COO (coordinate):** stores one (row, col, val) tuple per nonzero.

![[sparse-coo.png]]

- **CSR (compressed sparse row):** sorts nonzeros by row. Row i is `val[ptr[i] : ptr[i+1]]`.

![[sparse-csr.png]]

- **CSC (compressed sparse column):** CSR over columns. Column j is `val[ptr[j] : ptr[j+1]]`.

![[sparse-csc.png]]

### Matrix Multiplication Algorithms

$A \in \mathbb{R}^{M \times K}$, $B \in \mathbb{R}^{K \times N}$ and $C \in \mathbb{R}^{M \times N}$. For MV, $b \in \mathbb{R}^{K}$ and $c \in \mathbb{R}^{M}$. 

$$
\forall m \in [0, M),\quad c_m = \sum_{k=0}^{K-1} A_{m,k} \times b_k \tag{1}
$$

$$
\forall (m, n) \in [0, M) \times [0, N),\quad C_{m,n} = \sum_{k=0}^{K-1} A_{m,k} \times B_{k,n} \tag{2}
$$

```
Inner product                      Outer product                      Gustavson
for m in [0, M)                    for k in [0, K)                    for m in [0, M)
  for n in [0, N)                    for m in [0, M)                    for k in [0, K)
    for k in [0, K)                    for n in [0, N)                    for n in [0, N)
      C[m,n] += A[m,k] × B[k,n]          C[m,n] += A[m,k] × B[k,n]          C[m,n] += A[m,k] × B[k,n]
```

> [!example]- Inner product, step by step
> ![[spgemm-inner-steps.png]]

> [!example]- Outer product, step by step
> ![[spgemm-outer-steps.png]]

> [!example]- Gustavson, step by step
> ![[spgemm-gustavson-steps.png]]

*Source: [Isaac-Chassande et al., TACO 2024](https://dl.acm.org/doi/10.1145/3640542)*

## Hardware

Only Looking at MM as targeting ML Workloads, using CSR Format and Gust Algo. (TODO: Understand why choose these)
### MM

![[gemm-hardware.png|623]] ![[gemmini-architecture.png|666]]

*Left: simplified dataflow, generated with claude. Right: Gemmini architecture, from [Genc et al., DAC 2021](https://arxiv.org/abs/1911.09925).*

### SpMM

![[spmm-sextans-flow.png|631]] ![[sextans-architecture.png|660]]

*Left: simplified flow, based on Sextans. Right: Sextans architecture, from [Song et al., FPGA 2022](https://arxiv.org/abs/2109.11081).*

### SpMSpM

![[spmspm-gamma-flow.png|601]] ![[gamma-operation.png|670]]

*Left: simplified flow, based on Gamma. Right: Gamma's operation, from [Zhang et al., ASPLOS 2021](https://dspace.mit.edu/handle/1721.1/145981).*

- **Format:** A, B and C are all plain CSR (Gamma calls each compressed row a fiber), so there is no conversion between formats.
- **Scheduler:** walks the rows of A and turns each into a task for whichever PE is free. Rows of A with more than 64 nonzeros become a tree of merges.
- **FiberCache:** holds rows of B, fetched ahead of use, and partial output rows.
- **PE:** merges up to 64 rows of B by column, scales each element by its value from A, adds neighbours with the same column, and writes the finished row of C back in CSR.

## References
- [Dedicated Hardware Accelerators for Processing of Sparse Matrices and Vectors: A Survey](https://dl.acm.org/doi/10.1145/3640542)
- [Gemmini: Enabling Systematic Deep-Learning Architecture Evaluation via Full-Stack Integration (Genc et al., DAC 2021)](https://arxiv.org/abs/1911.09925)
- [Sextans: A Streaming Accelerator for General-Purpose Sparse-Matrix Dense-Matrix Multiplication (Song et al., FPGA 2022)](https://arxiv.org/abs/2109.11081)
- [Gamma: Leveraging Gustavson's Algorithm to Accelerate Sparse Matrix Multiplication (Zhang et al., ASPLOS 2021)](https://dspace.mit.edu/handle/1721.1/145981)
