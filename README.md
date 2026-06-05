# Surface Code on IQM Emerald — Challenge Overview

## Quantum Error correction - Memory experiment

Quantum computers are fundamentally noise-limited: every gate, every idle moment, every measurement introduces errors that accumulate faster than useful computation can proceed. 
Quantum error correction addresses this by encoding a single logical qubit across many physical qubits and continuously measuring stabilizer operators — parity checks that reveal error syndromes without collapsing the encoded information. 
Surface codes have emerged as the leading practical approach.
A surface code is defined, via the number of physical qubits it needs, the distance it covers, and the number of logical qubits. $$
They map directly onto the square-lattice connectivity that superconducting processors like IQM Emerald provide.
Still, writing out a whole a whole pipeline is notoriously hard. 
The challenge you will be facing today will have you operating on the forefront of science! 

## The Pipeline

You are building and completing a proof-of-concept quantum error correction pipeline that runs a surface code on real superconducting hardware and produces a logical error rate. The full chain is:

```
(Stim circuit  →)  Qiskit  →  IQM Emerald (Resonance)  →  Syndrome extraction  →  Decoder  →  LER
```

Each stage has a clear role. 

* Stim/ Qiskit defines the surface code circuit including stabilizer structure, qubit coordinates, detectors, and the logical observable. Stim is optional, but makes postprocessing a lot easier. 
* The Qiskit layer translates that abstract circuit into something the hardware can execute. 
* IQM Resonance runs the transpiled circuit and returns shot-by-shot measurement bitstrings. 
* Syndrome extraction converts those raw bitstrings into detection events, which are what the decoder actually needs, not the raw measurements themselves. 
* The decoder then uses those events to infer whether a logical error occurred.

In real hardware, you would then feed the information on logical errors back into the QPU and correct errors on the fly. But real hardware is limited, so this is something we cant do (yet). This is called a memory experiment.

**This pipeline is offline**. Decoding happens after all shots are collected. There is no requirement to feed correction results back to the QPU during coherence time.

---

## Judging criteria

1) Theoretical correctness. Does the experiment actually set up a surface correctly? Is the pipeline fully functional? (30%)
2) Sophistication of the implementation. Have you optimized your circuits for the performance/architecture of the quantum computer? Can you prove a low number of SWAPS? (30%)
3) Logical error rate statistics. (20%)
4) Flexibility of the experiment and integration of advanced functionalities - Circuit compilation techniques, Noise Model improvements, Pulse-level compilations,... (20%)


Bonus points will be awarded (at our discretion) for:
- Hardware flexibility.
- Integration of advanced functionalities.
- Experimentally identifying the most important reasons for improving LER.
- Other particularly cool and surprising ideas

## Submissions

Submit your code and any other key results through [this Google form](https://docs.google.com/forms/d/e/1FAIpQLSeKTmoGUzxhrc4bPtLJ5Vp3VtoFSrtoes1UP3_6IoLgZDLIxg/viewform?usp=publish-editor). 


# Useful Resources

**Please see the ``UsefulResources.md`` markdown for an overview over useful resources!** Such as, links to **IQM PulLA**, client-side libraries such as **Dynamical Decoupling**, access links to IQM Resonance!
**To access PulLA you need to be a part of a paid organisation, please approach your mentors about this!**


---

## What Is Already There

**All the provided functionality is optional!** Should you wish to build from scratch, feel free to do so! 

### Circuit translation (`surface_code.py`)

Should you wish to use Stim, the full Stim-to-Qiskit translation layer is implemented. Functionality:

* Stim uses sparse, non-contiguous qubit indices whereas Qiskit requires a contiguous `0..N-1` range, so the converter builds a dense re-indexing map. 
* strips all noise and annotation instructions (which exist only in Stim's simulation model and have no hardware equivalent), 
* handles mid-circuit reset by translating Stim's `MR` instruction into a measure-then-reset pair, 
* preserves measurement order exactly.

### Syndrome extraction (`extract_syndromes.py`)

The full syndrome extraction step is implemented using Stim's ``compile_m2d_converter()``, which converts raw measurement bitstrings into detection events and observable flips in a single call. If you are working within a pure Qiskit workflow, you can replace this with a manual implementation: detection events are the XOR of consecutive ancilla measurement rounds (a stabilizer that changes between rounds indicates an error), and the logical observable is recovered from the parity of the final data qubit measurements. Either way, what matters for the decoder is the same output shape — a (shots, num_detectors) boolean array of detection events and a (shots, num_observables) array of logical outcomes. The dictionary also carries per-shot syndrome weights and per-detector firing rates as diagnostics.

### Hardware execution (`run_on_hadware.py`)

The hardware connection boilerplate is complete. It connects to IQM Resonance via `IQMProvider`, retrieves the backend, transpiles the Qiskit circuit, submits the job, and converts the returned Qiskit counts dictionary into the numpy array format the syndrome extractor expects. The Qiskit bitstring reversal (Qiskit is right-to-left, Stim is left-to-right) is handled in the helper `counts_to_measurement_array`.
**Attention here!** Automatic SWAP insertion for pairs that aren't natively adjacent on the hardware grid! You can cirumvent this by writing your own stim/qiskit-to-emerald hardware mapping.

### Utilities (`inernal_helpers.py`)

Three helper functions support the pipeline, mostly for stim to qiskit and postprocessing functionality.

* `get_qubit_lists` classifies qubits as data or ancilla using the coordinate parity convention Stim uses for the rotated surface code. 
* `get_meas_order` returns qubit indices in the order they are measured to align hardware bitstrings with Stim's measurement record. 
* `counts_to_measurement_array` does the Qiskit-to-numpy conversion including the endianness correction. 

Note the docstrings reminding you, that you should verify this logic carefully before fully trusting it.

---

## What Needs To Be Built

### 1. A way to generate the surface code circuit

The pipeline currently has no entry point. You need to build a surface code generator in the framework of your choice! 

Concretely, you need a function that accepts:

* a code distance
* a number of stabilizer rounds, 
* which memory experiment we are looking at (Z or X)
* optionally a noise model, 
* and returns a `stim.Circuit` or a `qiskit.QuantumCircuit`, or ...

You do not have to build this in Stim or Qiskit if you prefer to use another framework! 

One decision worth thinking about: Most surface codes default to mid-circuit resets on ancilla qubits after a measurement (`MR`), so each round starts with a fresh ancilla and each raw measurement directly gives that round's syndrome. An alternative is to omit the reset: the ancilla then accumulates across rounds, and each raw measurement is the ``XOR`` of all syndromes up to that point. The detector definitions are identical in both cases, but the no-reset version avoids the reset error, leaving the ancilla live and decohering between rounds. Both are viable — the no-reset path is particularly relevant on hardware where resets introduce errors.

### 2. A way to decode the syndromes and compute the logical error rate

`decode_hardware_results` does not do anything in its current form. The detection events and observable flips are already correctly formatted when they arrive here — `det_events` is a `(shots, num_detectors)` boolean array, `obs_flips` is `(shots, num_observables)`. What is missing is the actual decoding step and the statistical aggregation into a logical error rate and its uncertainty.

Options are plenty, but PyMatching (specifically the MWPM decoder) will give you the fastest out-of-the-box result. 

### 3. (Optional but Recommended) A simulation baseline

There is no simulation functionality. Before spending QPU time, it is strongly advisable to run the same circuit through a simulator with a noise model and validate that the pipeline is working correctly end to end. A simulation baseline also gives you an expected LER to compare against the hardware result, even though there will be big discrepancies.

### 4. (Optional) Custom hardware qubit placement

`build_emerald_qubit_map` returns an empty dictionary. The Qiskit transpiler handles qubit placement automatically and will find a valid mapping, but it does so without knowledge of the surface code's structure. The rotated surface code can be aligned to Emerald's square grid, eliminating SWAP overhead entirely. If you are using Stim, the `print_qubit_map` function will then show you the mapping and flag any non-native pairs, so you can verify before submitting. In Qiskit this verification is straight forward.

---

## Extensions

### Pulse-level compilation with PulLA

The current pipeline operates at the gate level and relies on Qiskit's transpiler to map to the hardware's native gate set. A more direct path is to compile the stabilizer circuit to pulses using **PulLA**, IQM's pulse-level compilation framework. This bypasses the gate abstraction entirely and can produce significantly shallower circuits by fusing or reshaping pulses that would otherwise be separate gates. For a noise-limited experiment like this, reducing circuit depth directly translates to fewer errors before the decoder sees the syndrome, which is the most direct lever on logical error rate.
**Also activate advanced circuit compilation techniques, such as dynamical decoupling (DD), readily available within IQMs stack.**


### Different decoders and advanced functionalities

PyMatching (minimum-weight perfect matching) is the natural starting point, but it is not the only option. **Union-Find decoding** is asymptotically faster and nearly as accurate near threshold. **Belief propagation** decoders can incorporate more detailed noise information. The MWPM graph itself can be weighted by calibration data from Resonance, which typically improves decoding accuracy on real hardware. **There are many more, so feel to investigate!** 
In particular using calibration data and setting up experiments for extended noise model characterization are of very high value! These can massively improve the decoding pipeline. 

For a more substantial extension, the **NVIDIA Ising Predecoder** offers a hardware-accelerated decoding path. This route requires working through the Ising model's `MemoryCircuit` class for circuit generation (which handles native X-basis preparation and measurement and a different Stim-to-Qiskit translation), and is therefore more involved. Please approach us if you are interested! 

### Different hardware patches and code distances

The pipeline as set up targets a distance-3 rotated surface code (17 qubits) on a single 5×5 region of Emerald. Several natural extensions exist: running the same code on different 5×5 sub-regions of the chip and comparing logical error rates across regions reveals spatial variation in hardware quality. Trying distance-5 (49 qubits) would require a larger patch but fits on Emerald's full qubit count. Trying the distance-3 code on Garnet also gives you an idea on how a different QPU behaves. 




