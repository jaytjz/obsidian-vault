## Introduction

This is a wish list for a development workflow I would have preferred to use during my Qualcomm internship. The workflow is based on [OpenTitan's CI setup](https://opentitan.org/book/doc/contributing/ci/index.html).

> [!NOTE]
> The tools used at Qualcomm were Synopsys VCS, Verdi and UVM, the same as in Peter Cheung's course.

## Verification

In industry, UVM (Universal Verification Methodology) is used to write reusable testbench components. The relationship between SystemVerilog and UVM is like the one between JavaScript and React.

IMO, UVM is disgusting. It was born out of hardware engineers hacking OOP concepts onto Verilog, and I would much rather write testbenches in Python.

| Tool | Purpose |
| --- | --- |
| [cocotb](https://www.cocotb.org/) | Python testbenches that drive the simulator (supports VCS, Verilator, Icarus) |
| [pyuvm](https://github.com/pyuvm/pyuvm) | UVM reimplemented in Python on top of cocotb |
| [cocotb-coverage](https://github.com/mciepluc/cocotb-coverage) | Functional coverage and constrained random stimulus |
| [SymbiYosys](https://github.com/YosysHQ/sby) | Formal verification front end for Yosys |

## Open Source Simulators and Waveform Viewers

To use alongside VCS and Verdi/GTKWave.
### Simulators

| Tool | Purpose |
| --- | --- |
| [Verilator](https://www.veripool.org/verilator/) | Compiles SystemVerilog to C++. Very fast, good for CI and regressions. Two state, so it won't catch X propagation bugs |
| [Icarus Verilog](https://steveicarus.github.io/iverilog/) | Event driven and four state like VCS, but slower and with only partial SystemVerilog support |

### Waveform Viewers

| Tool | Purpose |
| --- | --- |
| [Surfer](https://surfer-project.org/) | Modern waveform viewer for VCD and FST files that also runs in the browser |
## Linters

They didn't use linters at Qualcomm :( so let's use:

| Tool | Purpose |
| --- | --- |
| [slang-server](https://github.com/hudson-trading/slang-server) | SystemVerilog language server with diagnostics, built on the slang compiler |
| [Ruff](https://docs.astral.sh/ruff/) | Python linter and formatter |

## CI and Build System

| Tool | Purpose |
| --- | --- |
| [GitHub Actions](https://docs.github.com/en/actions) | Run lint and tests on every PR |
| [Bazel](https://bazel.build/) | Cached, reproducible builds and test runs (OpenTitan uses it) |
| [Docker](https://www.docker.com/) | Pin tool versions so CI and local runs match |