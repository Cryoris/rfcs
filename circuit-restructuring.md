# Restructuring of Aqua's circuits 

## Current situtation

Aqua has a directory `qiskit/aqua/circuits` containing a set of circuits and gates used in algorithms or components. 
These are
```
qiskit/aqua/circuits
├── boolean_logical_circuits.py
├── fixed_value_comparator.py
├── fourier_transform_circuits.py
├── gates
│   ├── boolean_logical_gates.py
│   └── multi_control_multi_target_gate.py
├── linear_rotation.py
├── phase_estimation_circuit.py
├── piecewise_linear_rotation.py
├── polynomial_rotation.py
├── statevector_circuit.py
└── weighted_sum_operator.py
```

A lot of the functionality in there is general enough to be migrated to Terra, such as the gates or any arithmetic operations. 
The leftover circuits are algorithm-specific and could be moved to the algorithm folder for clarity, rendering the `circuits` 
directory obsolete. 
Removing this `circuits` folder would also be in favour of streamlining Terra and Aqua, since Terra lately introduced 
`qiskit/circuits/library` for a family of ready-to-use circuits.

## The content of `circuits`

The circuits can be grouped as follows.

### Gates

As the multi-controlled gates, these should be moved to Terra under `qiskit/extensions/standard`,
since they monkey-patch gates to the `QuantumCircuit`.

```
boolean_logical_gates multi_controlled_multi_target_gate
```

*Suggestion: Move to `qiskit/extensions/standard`.*

### Arithmetic circuits

Arithmetic circuits do mathematical operations on qubit states or their amplitude.

```
linear_rotation piecewise_linear_rotation polynomial_rotation fixed_value_comparator weighted_sum_operator
```

These operations are all implemented as `CircuitFactory` and are used in the context of quantum amplitude estimation.

Generic arithmetic operations such as these should probably be moved to Terra, since they are basic operations.
Possibly the circuit library directory is a suitable place, potentially under a `arithmetic` subfolder.
With the deprecation of the `CircuitFactory` we have two options for further refactoring these classes:

1. Derive them from the `QuantumCircuit`. All information is present in the initializer anyways and the 
  functionality is maintained by converting the circuit to a gate.
2. Implement them in the same fashion as the Ansatz, i.e. allow late initialization of the attributes. 
  The functionality is maintained also by calling `to_gate` and using its methods.

Another possibility that should be mentioned:
Since these factories are only used in amplitude estimation, they could also go in the amplitude estimation directory.

*Suggestion: Move to `qiskit/circuits/library/arithmetic` and, later, 
refactor the classes to derive from the `QuantumCircuit`.*


### Component or algorithm specific circuits

These are circuits specifially created for a certain algorithm or component in Aqua.
```
phase_estimation_circuit: algorithm/QuantumPhaseEstimation
boolean_logical_circuits: components/oracles
fourier_transform_circuit: components/qft components/iqft
statevector_circuit: components/initial_state (also used in feature_map however)
```

#### Phase estimation

This circuit implements the phase estimation algorithm circuit.

*Suggestion: Move to the `qiskit/aqua/algorithms/minimum_eigen_solvers`.*

#### Boolean logical circuits

These circuits are only used for oracle creation, e.g. used in the context of Grover's algorithm.

*Suggestion: Move to `qiskit/aqua/components/oracles`.*

#### Fourier transform circuit

This circuit implements the Fourier transform and, based on a keyword argument, it's inverse.

The Fourier transformation is a central operation in many algorithms and could be moved to Terra. 
Conceptually, it would fit in the circuit library directory. 
However it must be checked if it is useful to implement the Fourier transform as `QuantumCircuit`, since we would 
like to change the number of qubits in retrospective. 

Since only used in the `qft/iqft` component, the Fourier transform circuit could also be moved to `qiskit/aqua/components/qft`.

*Suggestion: Slight preference for `qiskit/circuits/library`, but can also remain in Aqua.*

#### Statevector circuit

This class is essentially a wrapper around the `initialize` instruction (which should probably be replaced by `isometry`, 
which seems to be more efficient, @ajavadi?). 

Since no further action is done, we can probably deprecate the entire class and update the `InitialState` and `RawFeatureVector`
to directly use the according `QuantumCircuit` method.

*Suggestion: Deprecate in favour of using `initialize/isometry`.*









