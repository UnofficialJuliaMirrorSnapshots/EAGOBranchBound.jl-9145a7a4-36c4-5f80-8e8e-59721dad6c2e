# EAGOBranchBound.jl
A branch-and-bound library for Julia

[![Build Status](https://travis-ci.org/MatthewStuber/EAGOBranchBound.jl.svg?branch=master)](https://travis-ci.org/MatthewStuber/EAGOBranchBound.jl)
[![Coverage Status](https://coveralls.io/repos/github/MatthewStuber/EAGOBranchBound.jl/badge.svg?branch=master)](https://coveralls.io/github/MatthewStuber/EAGOBranchBound.jl?branch=master)
[![codecov.io](http://codecov.io/github/MatthewStuber/EAGOBranchBound.jl/coverage.svg?branch=master)](http://codecov.io/github/MatthewStuber/EAGOBranchBound.jl?branch=master)

[![](https://img.shields.io/badge/docs-stable-blue.svg)](https://MatthewStuber.github.io/EAGO.jl/stable)
[![](https://img.shields.io/badge/docs-latest-blue.svg)](https://MatthewStuber.github.io/EAGO.jl/latest)

## Authors
- [Matthew Wilhelm](httppsor.uconn.eduour-team), Department of Chemical and Biomolecular Engineering,  University of Connecticut (UCONN)

## Table of Contents
- [**Installation**](#installation)
- [**Capabilities**](#capabilities)
- [**Usage Instructions**](#usage)
  - **Example 1** - Setup and Solve a Basic Problem
  - **Example 2** - Adjust Tolerances
  - **Example 3** - Select Alternative Search Routines
  - **Example 4** - Adjust Information Printed

## Installation
To install the package, from within Julia do

```julia
julia> Pkg.add("EAGOBranchBound")
```

## Capabilities
This package is meant to provide a flexible framework for implementing branch-and-bound based optimization routines in Julia. All components of the branch-and-bound routine can be customized by the individual user: lower bounding problem, upper bounding problem, .
## Usage

### Package Structure
The package consists of a main solve algorithm that executes as depicted in the flowchart below. Routines for setting the objects to implement standard B&B routines are also provided using a `set_default!()` function.

![BnBChart](https://github.com/MatthewStuber/EAGOBranchBound.jl/blob/master/docs/BnBChart1.jpg)

- The preprocessing routine has inputs `(feas,X,UBD,k,d,opt)` and outputs `feas::Bool,X::Vector{Interval{Float64}}`. The initial feasibility flag is `feas`, the bounds on the variables are `X`, the current upper bound is `UBD`, the iteration number is `k`, the node depth is `d`, and a solver option storage object is `opt`.
- The lower bounding routine has inputs `(X,k,d,opt,UBDg)` and provides outputs `(val,soln,feas,Lsto)`. The value of the subproblem is `val`, the solution of the subproblem is `soln`, it's feasibility is `feas`, and `Lsto` is a problem information storage object.
- The upper bounding routine has inputs `(X,k,d,opt,UBDg)` and provides outputs `(val,soln,feas,Usto)`. he value of the subproblem is `val`, the solution of the subproblem is `soln`, it's feasibility is `feas`, and `Uto` is a problem information storage object.
- The postprocessing routine has inputs `(feas,X,k,d,opt,Lsto,Usto,LBDg,UBDg)` and outputs `feas::Bool,X::Vector{Interval{Float64}}`.
- The repeat check has inputs `(s,m,X0,X)` where `s::BnBSolver` is a solver object, `m::BnBModel` is a model object, `X0::Vector{Interval{Float64}}` are node bounds after preprocessing, and `X::Vector{Interval{Float64}}` are the node bounds generated after postprocessing. Returns a boolean.
- The bisection function has inputs `(s,m,X)` where `s::BnBSolver` is a solver object, `m::BnBModel` is a model object, and `X::Vector{Interval{Float64}}` is the box to bisect. It returns two boxes.
- The termination check has inputs `(s,m,k)` where `s::BnBSolver` is a solver object, `m::BnBModel` is a model object, and `k::Int64` is the iteration number. Returns a boolean.
- The convergence check has inputs `(s,UBDg,LBD)` where `s::BnBSolver` is a solver object, `UBDg` is the global upper bound, and `LBD` is the lower bound.

### Example 1 - Setup and Solve a Basic Problem
In the below example, we solve for minima of **f(x)=x<sub>1</sub>+x<sub>2</sub><sup>2</sup>** on the domain [-1,1] by [2,9]. Natural interval extensions are used to compute the upper and lower bounds. The natural interval extensions are provided by the Validated Numerics package.

First, we create a BnBModel object which contains all the relevant problem info and a BnBSolver object that contains all nodes and their associated values. We specify default conditions for the Branch and Bound problem. Default conditions are a best-first search, relative width bisection, normal verbosity, a maximum of 1E6 nodes, an absolute tolerance of 1E-6, and a relative tolerance of 1E-3.
```julia
using EAGOBranchBound
using ValidatedNumerics
b = [Interval(-1,1),Interval(1,9)]
a = BnBModel(b)
c = BnBSolver()
EAGOBranchBound.set_to_default!(c)
c.BnB_atol = 1E-4
```
Next, the lower and upper bounding problems are defined. These problems must return a tuple containing the upper/lower value, a point corresponding the upper/lower value, and the feasibility of the problem. We then set the lower/upper problem of the BnBModel object and solve the BnBModel & BnBSolver pair.
```julia
function ex_LBP(X,k,pos,opt,temp)
  ex_LBP_int = @interval X[1]+X[2]^2
  return ex_LBP_int.lo, mid.(X), true, []
end
function ex_UBP(X,k,pos,opt,temp)
  ex_UBP_int = @interval X[1]+X[2]^2
  return ex_UBP_int.hi, mid.(X), true, []
end

c.Lower_Prob = ex_LBP
c.Upper_Prob = ex_UBP

outy = solveBnB!(c,a)
```
The solution is then returned in b.soln and b.UBDg is it's value. The corresponding output displayed to the console is given below.

![BnB_Output](https://github.com/mewilhel/Julia_BnB/blob/master/Documentation/src/BnB_Output.png)

### Example 2 - Adjust Solver Tolerances
The absolute tolerance can be adjusted as shown below
```julia
julia> a.BnB_tol = 1E-4
```
The relative tolerance can be changed in a similar manner
```julia
julia> a.BnB_rtol = 1E-3
```
### Example 3 - Select Alternative Search Routines
In the above problem, the search routine could be set to a breadth-first or depth-first routine by using the set_Branch_Scheme command
```julia
julia> EAGOBranchBound.set_Branch_Scheme!(a,"breadth")
julia> EAGOBranchBound.set_Branch_Scheme!(a,"depth")
```
### Example 4 - Adjust Information Printed
In order to print, node information in addition to iteration information the verbosity of the BnB routine can be set to full as shown below
```julia
julia> EAGOBranchBound.set_Verbosity!(a,"Full")
```
Similiarly, if one wishes to suppress all command line outputs, the verbosity can be set to none.
```julia
julia> EAGOBranchBound.set_Verbosity!(a,"None")
```

## Branch-and-Bound Background
The majority of deterministic global optimization techniques use a branch-and-bound framework (or the closely related branch-and-reduce framework). Branch-and-bound search algorithms operate by partitioning a compact parameter space into a series of non-overlapping spaces (termed nodes). Upper and lower bounds of the function on a node are estimated, then compared with a global upper and lower bound. Any node that is found with an lower bound below the global upper bound is discarded (fathomed).

## References
- Floudas, Christodoulos A. Deterministic global optimization: theory, methods and applications. Vol. 37. Springer Science & Business Media, 2013.
- Horst, Reiner, and Hoang Tuy. Global optimization: Deterministic approaches. Springer Science & Business Media, 2013.

## Related Packages
- [ValidatedNumerics.jl](https://github.com/JuliaIntervals/ValidatedNumerics.jl), a Julia library for validated interval calculations, including basic interval extensions, constraint programming, and interval contactors
- [IntervalOptimisation.jl](https://github.com/JuliaIntervals/IntervalOptimisation.jl), implements a branch-and-bound type global optimization routine using validated interval arithmetic
