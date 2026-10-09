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

- **Dataflow:** DMA loads A and B tiles from DRAM into the scratchpad, the systolic array computes a C tile into the accumulator, and DMA writes C back.
- **Array (output stationary):** A flows right and B flows down, skewed by one cycle per row or column. Each PE keeps one element of C.
- **PE:** one MAC per cycle, c += a × b. a and b are registered, then passed to the next PE.

### SpMM

![[spmm-sextans-flow.png|631]] ![[sextans-architecture.png|660]]

*Left: simplified flow, based on Sextans. Right: Sextans architecture, from [Song et al., FPGA 2022](https://arxiv.org/abs/2109.11081).*

- **Dataflow:** each PEG has its own Read A stream of nonzeros (m, k, a). Read B loads a K0 × N0 window of B, which is passed from PEG to PEG. C accumulates on chip, then Collect C scales it by α and Comp C adds βC_in, so Sextans computes C = αAB + βC.
- **Array:** 8 PEGs of 8 PEs. Row m goes to PE m mod 64, so PEs own disjoint rows of C and never write the same element.
- **PE:** for each nonzero, it reads row k of its local B copy and does N0 MACs, C[m, j] += a × B[k, j], into its C scratchpad.
- **Hazards:** two nonzeros for the same row back to back would wait on the adder latency, so Sextans reorders nonzeros in preprocessing to keep issuing one per cycle.

### SpMSpM

![[spmspm-hardware.png|615]] ![[matraptor-architecture.png|656]]

*Left: simplified dataflow, based on MatRaptor. Right: MatRaptor architecture, from [Srivastava et al., MICRO 2020](https://www.csl.cornell.edu/~zhiruz/pdfs/matraptor-micro2020.pdf).*

- **Lanes:** each lane has a sparse A loader (SpAL), a sparse B loader (SpBL) and a PE, on its own HBM channel. Rows of A are dealt round robin across lanes.
- **Loading:** SpAL streams row i of A as (a, i, k). For each one, SpBL streams row k of B and sends (a, b, i, j) to the PE.
- **PE:** Phase I multiplies and merges each partial row into a queue sorted by column. Phase II picks the smallest column across the queues, sums matching entries with an adder tree, and streams the finished row of C out.
- **Overlap:** two queue sets are double buffered, so Phase II drains row i while Phase I builds row i + 1. C is written in C²SR, where each row's channel is fixed, so lanes never wait for each other.

## References
- [Dedicated Hardware Accelerators for Processing of Sparse Matrices and Vectors: A Survey](https://dl.acm.org/doi/10.1145/3640542)
- [Gemmini: Enabling Systematic Deep-Learning Architecture Evaluation via Full-Stack Integration (Genc et al., DAC 2021)](https://arxiv.org/abs/1911.09925)
- [Sextans: A Streaming Accelerator for General-Purpose Sparse-Matrix Dense-Matrix Multiplication (Song et al., FPGA 2022)](https://arxiv.org/abs/2109.11081)
- [MatRaptor: A Sparse-Sparse Matrix Multiplication Accelerator Based on Row-Wise Product (Srivastava et al., MICRO 2020)](https://www.csl.cornell.edu/~zhiruz/pdfs/matraptor-micro2020.pdf)
