# DiRAC 4 Performance profile

- Author(s): 
- Application:
- Related DiRAC project(s):

*Please use this repository by clicking [this link](https://github.com/new?template_name=dirac4-performance-profile&template_owner=aportelli) or the "Use this template" button on GitHub. Please fill the sections below, and add any supporting evidence as links to code or files directly in your clone of this repository. All guidelines in italic can be removed from the final document.*

## 1. Performance-critical routines
*Please list here representative routines that dominate a production run. Please provide links and references to code locations of these routines, as well as profiler evidence that they are indeed dominant. This should be understood as a best-effort analysis to capture most of the time enveloppe for your typical runs, rather than an exhaustive study. Profiler evidence can be from a third-party profiler, as well as custom instrumentation; for the latter please describe succintly how profiling is implemented.*

## 2. Theoretical performance expectations
*For all routines identified in the previous section, please provide theoretical expectations on how well the routine is expected to perform in a best-case scenario. "Performance" is understood here as absolute figures or formulas that can be used to constrain hardware specification (e.g. GiB/s, GFlop/s but **not** time-to-solution or speedup relatively to a reference architecture). For example, if the [Roofline Model](https://en.wikipedia.org/wiki/Roofline_model) is an appropriate description for a given routine, then providing the arithmetic intensity of the routine is sufficient.*

## 3. Benchmark applications
*Please provide references to the source code of benchmark applications measuring the performances figures described in Sec. 2.*

## 4. Example of benchmark result
*Please provide examples of results obtained from the benchmark described in the previous section. These example are made to be representative of an environment suitable for production, and contributors should not hesitate to choose a favourite, well-known system to obtain these results. Please comment on how well the benchmark results match the theoretical expectations in Sec. 2. If large discrepancies are observed, please comment on your current understanding of the root cause, as well as resulting opportunities for future optimisation.*
