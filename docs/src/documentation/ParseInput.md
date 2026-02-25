# Inputs to qdsim

A simulation depends upon the specification of the system. This is done in the `system` TOML file, which specifies three different things:
- the `units` in use
- the Hamiltonian for the problem being simulated:
    - the description of the `system`
    - the description of the `bath` or environment

Along with this `system` TOML file, a `simulation` TOML file also needs to be prepared that provides the details of the simulation.

## System File
Each of the three sections in the `system` file has a dedicated function for parsing the data. Before giving a full example of a `system` input file, we discuss the parameters accepted by the various sections.

### Specifying the Units
```@docs
QuantumDynamicsCLI.ParseInput.parse_unit
```

### Specifying the Hamiltonian
The basic problem under study has a general form, $$\hat{H} = \hat{H}_0 + \hat{H}_\text{env}$$, where $\hat{H}_0$ forms the system Hamiltonian and the $\hat{H}_\text{env}$ is the environment or bath Hamiltonian.

#### System Hamiltonian
```@docs
QuantumDynamicsCLI.ParseInput.parse_system
```

#### Bath Hamiltonian
```@docs
QuantumDynamicsCLI.ParseInput.parse_bath
```

```@docs
QuantumDynamicsCLI.ParseInput.get_bath
```

## Simulation File
The simulation file has only a single TOML section `[simulation]`. Every simulation file should have a `name` to identify the simulation and an `output` file that specifies an HDF5 file for storing all the data.

There are broadly three types of simulations that are currently supported:
- `dynamics` simulations that simulate the non-equilibrium dynamics of the given problem
- `equilibrium_rho` simulations that simulate the equilibrium density at the given temperature
- `complex_corr` simulations for calculating equilibrium correlation functions of various flavors
These are specified in the `calculation` field which, if unspecified, is taken to be `dynamics` by default.

The rest of the parameters required for a simulation file are specific to the type of simulation being run.

### Dynamics Simulations
Various methods of simulation are supported:
- Path Integral Methods using Feynman-Vernon Influence Functional[feynmanTheoryGeneralQuantum1963](@cite):
    - Quasi-adiabatic Propagator Path Integrals (QuAPI) [makriTensorPropagatorIterativeI1995, makriTensorPropagatorIterativeII1995](@cite)
    - Blip QuAPI [makriBlipDecompositionPath2014](@cite)
    - Time-Evolved Matrix Product Operators (TEMPO) [strathearnEfficientNonMarkovianQuantum2018](@cite)
    - Pairwise-Connected Tensor Network Path Integral (PC-TNPI) [bosePairwiseConnectedTensor2022](@cite)
- Hierarchical Equations of Motion (HEOM) [tanimuraNumericallyExactApproach2020](@cite)
- Generalized Quantum Master Equation (GQME) [nakajimaQuantumTheoryTransport1958, zwanzigIdentityThreeGeneralized1964](@cite)
- Multichromophore Incoherest Forster Resonance Energy Transfer [forsterZwischenmolekulareEnergiewanderungUnd1948, jangMultichromophoricForsterResonance2004](@cite)
- Bloch-Redfield Master Equation
- Transfer Tensor Method (TTM) [cerrilloNonMarkovianDynamicalMaps2014](@cite) coupled with any of the path integral methods
- Mapping Hamiltonian based semiclassical methods:
	- Quasiclassical / Linearized Semiclassical dynamics (LSC)
	- Partial Linearized Density Matrix dynamics (PLDM)
	- Spin-mapped version of LSC (Spin-LSC)
	- Spin-mapped version of PLDM (Spin-PLDM)

All of these dynamics methods require some core common parameters and then more specfic method-dependent parameters. The core parameters of all the dynamics methods are:
- `dt`: for the time-step in the units specified in the system file
- `nsteps`: for the number of steps of simulation of the dynamics
- `rho0`: the initial reduced density matrix
- `outgroup`: where to store the computed density matrix in the HDF5 file (i.e., the group name)

#### Specifying the initial density matrix
The simplest way to specify the initial density matrix is to set the `rho0` parameter to a file which will be parsed as a matrix. However, convenient shortcuts exist to specify most commonly used values of `rho0` as shown in the docstring of [QuantumDynamicsCLI.ParseInput.parse_operator](@ref).

```@docs
QuantumDynamicsCLI.ParseInput.parse_operator
```

#### Feynman-Vernon Influence Functional Simulations
There are two ways of incorporating the effect of non-Markovian memory in path integral simulations: iterative propagation beyond memory or using the transfer tensor method. To use TTM, one can choose one of the following:
- `method = "QuAPI-TTM"`: for using QuAPI within memory
- `method = "Blip-TTM"`: for using Blip within memory
- `method = "TEMPO-TTM"`: for using TEMPO within memory

Finally, if one does not intend on using TTM, we suggest using TEMPO for accessing long memory lengths efficiently. This is chosen by setting `method = "TEMPO"`.

In this family of methods, the non-Markovian memory is incorporated explicitly by specifying the number of time-steps it spans. For all the TTM-based methods, this memory length is set through the parameter, `rmax`. For `method = "TEMPO"`, it is set through the parameter, `kmax`.

Every method has a separate set of parameters that handle the balance between
accuracy and efficiency. In case of "QuAPI-TTM", that parameter is called
`cutoff` and it is by default set to $10^{-10}$. All paths with absolute value
of amplitude below this cutoff are ignored.

In the case of "Blip-TTM", the accuracy is set via the maximum number of blips
allowed, `max_blips`. If this is set to $-1$, that means all blips are included.

For "TEMPO-TTM", the accuracy is set through the tensor network evaluation
parameters. The `cutoff` of the singular value decomposition, and the maximum
bond dimension, `maxdim`, used in obtaining the matrix product state
representation of the path amplitude tensor are the two parameters that are used
for controlling the accuracy. The default values of these two parameters are
$10^{-10}$ and $1000$ respectively.

#### Semiclassical Simulations
There are principally two mapping Hamiltonian based semiclassical methods that one can choose: the Meyer-Miller-Stock-Thoss mapping based linearized semiclassics and PLDM, and the spin-mapping based versions of these. To use these methods, say `method = "$METHOD"` where `$METHOD` is one of:
- `LSC` or `PLDM` for the fully or partially linearized semiclassical dynamics using MMST mapping for the system degrees of freedom
- `Spin-LSC` or `Spin-PLDM` for the corresponding spin-mapped variants

As these methods perform a Monte-Carlo average to calculate the reduced density matrix, the following parameters must be set for all these methods:
- `num_bins`: the number of independent bins to calculate the average and standard deviation of the observables with
- `num_mc`: the number of trajectories for each bin

Apart from this, the spin-mapped semiclassical methods offer the choice of choosing the corresponding Stratonovich--Weyl kernel to use for transformation of the Hamiltonian of the system and the initial (reduced) density matrix. This is set by the `SW_transform` keyword, and the supported values are `QTransform`, `PTransform` and `WTransform`. By default, Spin-LSC uses `QTransform` and Spin-PLDM uses `WTransform`. **NOTE:** Spin-PLDM performs poorly with P and Q Stratonovich--Weyl kernels.

Focused initial sampling is only supported by Spin-LSC at the moment. Moreover, only `rho0` of the form $\ket{n}\bra{n}$ are supported currently. Focused sampling may be enabled for such initial reduced density matrices and Spin-LSC by setting the `focused_sampling` parameter to `true`.

These methods _must_ specify the number of discrete oscillators for each `bath` mode via the `num_osc` parameter as specified in the [Bath Hamiltonian](@ref) section above.
