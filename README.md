# DiRAC 4 Performance profile

- Author(s): Antonin Portelli
- Application: [Grid library](https://github.com/paboyle/Grid)
- Related DiRAC project(s): DP391, DP392, DP393

## 1. Performance-critical routines
Grid is a C++ library providing high-performance routines for lattice QCD calculations. Lattice QCD HPC applications can be quite heterogeneous, and are often implemented using additional frameworks reliant on Grid (e.g. the [Hadrons](https://github.com/aportelli/Hadrons) library). However, the time spent in most lattice QCD workflows is dominated by solving the Dirac equation, a sparse linear system given by the Dirac operator. This operator is not unique, and various scientific collaborations use different discretisations.

The Dirac operator is implemented using a [stencil](https://en.wikipedia.org/wiki/Stencil_(numerical_analysis)) approach, and parallelised across nodes by decomposing the physical domain equally into local volumes. Inverting the Dirac operator is done using iterative algorithms such as the [conjugate gradient](https://en.wikipedia.org/wiki/Conjugate_gradient_method) algorithm. The execution cost of algorithms of this class is generally dominated by multiplications with the matrix being inverted. In conclusion, lattice QCD applications are generally dominated by the cost of multiplying by a chosen Dirac operator, and therefore the implementation of such sparse matrix is one of the most performance-critical routines.

As an example, we provide the log of a production-scale time profile from one of the production applications used in the DP391 project. This application is based on the Hadrons library. This framework allows users to create heterogeneous workflows as a graph of elementary operations called *modules*. Hadrons has an internal profiler which accounts for the total time spent in each executed module. The [full log](iblep.log) is available as part of this repository, and we summarise the relevant profiles below.

The example job executed a total of 14974 modules, and the execution time per module type is given by
```plain
Hadrons : Message  : 99225.948296 s : ................ Module type breakdown
Hadrons : Message  : 99225.948315 s :                         Grid::Hadrons::MFermion::GaugeProp: 5.788e+04 s (58.4%)
Hadrons : Message  : 99225.948320 s :                        Grid::Hadrons::MFermion::ZGaugeProp: 3.196e+04 s (32.2%)
Hadrons : Message  : 99225.948325 s :                          Grid::Hadrons::MFermion::EMLepton: 3327 s (3.4%)
Hadrons : Message  : 99225.948330 s :                  Grid::Hadrons::MContraction::WardIdentity: 1774 s (1.8%)
Hadrons : Message  : 99225.948334 s :         Grid::Hadrons::MIO::LoadCoarseFermionEigenPack250F: 1652 s (1.7%)
Hadrons : Message  : 99225.948338 s :             Grid::Hadrons::MContraction::WeakMesonDecayKl2: 1517 s (1.5%)
Hadrons : Message  : 99225.948342 s :                         Grid::Hadrons::MContraction::Meson: 536.9 s (0.5%)
```
One can observe that 90.6% of the time is consumed by modules of the type `GaugeProp` and `ZGaugeProp`. Both modules are based on the same [template implementation](https://github.com/aportelli/Hadrons/blob/develop/Hadrons/Modules/MFermion/GaugeProp.hpp) in Hadrons. An example time profile for the execution of one such module is
```
Hadrons : Message  : 18824.480329 s : ................ Timings
Hadrons : Message  : 18824.480385 s : * GLOBAL TIMERS
Hadrons : Message  : 18824.480389 s :     total: 547 s (100.0%)
Hadrons : Message  : 18824.480424 s : execution: 538 s (98.3%)
Hadrons : Message  : 18824.480428 s :     setup: 9.069 s (1.7%)
Hadrons : Message  : 18824.480433 s : * CUSTOM TIMERS
Hadrons : Message  : 18824.480435 s :           Solver: 508.1 s (92.9%)
Hadrons : Message  : 18824.480440 s :   Import sources: 22.65 s (4.1%)
Hadrons : Message  : 18824.480443 s : Export solutions: 7.231 s (1.3%)
```
where the solver is observed to be by far the dominant contribution, as expected from the discussion above.

## 2. Theoretical performance expectations
For the sake of simplicity, we make the following assumptions here:
- we will only consider the even-odd component of the domain-wall fermion (DWF) sparse matrix, which is the main matrix used by DP391-DP393;
- we assume FP32 (single precision) operations only (the dominant cost in production since most workflows use mixed-precision solvers);
- we will assume each MPI process holds a local hypercubic four-dimensional volume with $N_L$ sites in each dimension.

Additionally, the DWF matrix has a size parameter denoted $L_s$, which is an $\mathcal{O}(10)$ integer. We will use the symbol $\mathrm{B}$ for bytes and $\mathrm{Flop}$ for floating-point operations. One application of the DWF matrix requires:
1. the execution of $F=N_L^4L_s\times 672~\mathrm{Flop}$ by the processor (CPU or GPU);
2. reading and writing $S_{\mathrm{int}}=N_L^4L_s\times 96~\mathrm{B}$ from the process memory;
3. sending and receiving $S_{\mathrm{halo}}=N_L^3L_s\times 384~\mathrm{B}$ through communications with other MPI processes (*halo exchange*).

A key aspect of the three operations above is that item 2 requires access to significantly more data than item 3; however, memory bandwidth is typically significantly higher than MPI communication bandwidth. In summary, the DWF matrix is generally a bandwidth-bound operation, and critically depends on fast memory and communications.

Theoretical predictions for the DWF matrix performance can be formulated using a [roofline model](https://en.wikipedia.org/wiki/Roofline_model) with two bandwidth bounds for memory and communication. The arithmetic intensities are given by
- $I_{\mathrm{mem}}=F/S_{\mathrm{int}}=7~\mathrm{Flop}/\mathrm{B}$ for local memory accesses;
- $I_{\mathrm{MPI}}=F/S_{\mathrm{halo}}=N_L\times 1.75~\mathrm{Flop}/\mathrm{B}$ for MPI communications.

Let us give a concrete example of expected performance for Tursa nodes based on 4 A100-80 SXM NVIDIA GPUs and 4 HDR200 network interfaces. Using [NVIDIA's official datasheet](https://www.nvidia.com/en-gb/data-center/a100/):
- each GPU has a peak FP32 performance of $P=19.5~\mathrm{TFlop}/\mathrm{s}$;
- each GPU has a peak memory bandwidth of $B_{\mathrm{mem}}=2039~\mathrm{GB}/\mathrm{s}$;
- GPUs within a node can communicate using NVLink with a peak bidirectional bandwidth of $B_{\mathrm{NVLink}}=600~\mathrm{GB}/\mathrm{s}$;
- the peak bidirectional network bandwidth between two nodes is $B_{\mathrm{net}}=200~\mathrm{GB}/\mathrm{s}$ (aggregate for four HDR200 interfaces per node).

Let us assume that $N_L=24$, which is one of the key values used in Grid benchmarks. One can check from the quantities above that the DWF matrix multiplication is systematically in the bandwidth-bound regime of the roofline model. Specifically:
- on a single GPU, the maximum allowed performance is given by $I_{\mathrm{mem}}B_{\mathrm{mem}}=15.3~\mathrm{TFlop}/\mathrm{s}$;
- on a single node with 4 GPUs, the maximum allowed performance from NVLink communications is $I_{\mathrm{MPI}}B_{\mathrm{NVLink}}=27.1~\mathrm{TFlop}/\mathrm{s}$ per GPU. Therefore, the bottleneck is still the memory bandwidth as in the previous case, and the total maximum node performance is expected to be $4I_{\mathrm{mem}}B_{\mathrm{mem}}=61.3~\mathrm{TFlop}/\mathrm{s}$;
- on multiple nodes, in the worst-case scenario all MPI processes have to communicate using the network, resulting in the expected performance of $I_{\mathrm{MPI}}B_{\mathrm{net}}=9~\mathrm{TFlop}/\mathrm{s}$ per node.

## 3. Benchmark applications
*Please provide references to the source code of benchmark applications measuring the performances figures described in Sec. 2.*

## 4. Example of benchmark result
*Please provide examples of results obtained from the benchmark described in the previous section. These example are made to be representative of an environment suitable for production, and contributors should not hesitate to choose a favourite, well-known system to obtain these results. Please comment on how well the benchmark results match the theoretical expectations in Sec. 2. If large discrepancies are observed, please comment on your current understanding of the root cause, as well as resulting opportunities for future optimisation.*
