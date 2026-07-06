# Appendix
> This Markdown page contains details of Formal Verification for CADA. Because zenodo can not compile markdown, for a better experience please download all files from repo and then see this markdown file locally.


## Formal Verification for Security Properties

Formal verification of CADA's core security properties P1–P3 and of its measurement features P4 is essential because CADA operates under a threat model in which an adversary can fully compromise the prover program on a resource-constrained MSP430 device. The trustworthiness of remote attestation relies on prover–TCM isolation and the integrity and atomicity of measurement operations. Without these guarantees, a compromised prover could read or modify measurement results or key sets (violating P1), interrupt or tamper with TCM operations (violating P2), or forge evidence without executing genuine measurement code (violating P3). Such violations would let the adversary evade both control-flow and data-flow attestations.

To formally assure these properties, we develop two SMV models—`cada_goto.smv` and `cada_read.smv`—and use nuXmv to verify the stated properties. The models abstract CADA's memory regions (persistent memory, volatile memory, and memory-mapped I/O), access policies (`APr`, `APw`, and `APx`), execution states (prover, TCM, and terminated), and critical actions (trampoline calls and evidence sending). Non-overlapping memory regions and access-control policies model CADA's isolation mechanisms. The properties are specified as LTL formulas that encode temporal guarantees for isolation, integrity, and atomicity.

**Verified Security Properties and LTL Formulas**

| Property | Formula ID | LTL Formula | Specification |
|---|---|---|---|
| P1 | F1 | `G(exec_state = prover -> !in_tcm_data)` | Prover isolation from TCM data |
| P1 | F2 | `G((exec_state = prover & dma_active) -> !DMA_APw)` | Prover cannot use DMA write |
| P1 | F3 | `G((exec_state = prover & (utdm_b <= i & i <= utdm_e)) -> !APw)` | Prover cannot write the evidence and key set |
| P1 | F4 | `G(evidence_sent -> sent_through_hook)` | Prover can only send evidence through the TCM hook function |
| P1 | F5 | `G((utc_b <= i & i <= utc_e) -> !APw)` | Write protection from the prover on trampoline code and hook function |
| P2 | F6 | `G((in_trampoline & APx) -> G(APx))` | Trampoline-code runtime atomicity |
| P2 | F7 | `G(trampoline_call -> G(state != end))` | Trampoline cannot be interrupted |
| P3 | F8 | `G(trampoline_call -> prover_instrumented)` | Prover can generate evidence only if instrumented correctly |
| P3 | F9 | `G(prover_executing -> prover_instrumented)` | Ensures prover instrumentation |
| P4 | F10 | `G(in_prover_interval -> measure_now)` | Continuous measurements |
| P4 | F11 | `G(ctrl_event -> X ctrl_flow_measured)` | Circumvention-resilient measurements |
| P4 | F12 | `G(prover_data_write -> X mem_write_measured)` | Circumvention-resilient measurements |
| P4 | F13 | `G(prover_data_read -> X mem_read_measured)` | Circumvention-resilient measurements |
| P4 | F14 | `G(measure_now -> (ctrl_flow_measured & mem_write_measured & mem_read_measured))` | Multi-faceted execution-integrity measurements |

The `cada_read.smv` model focuses on memory access control and defines the control-flow predicates `in_prover_interval`, `ctrl_event`, and `ctrl_flow_measured`, as well as the aggregate flag `measure_now`. It verifies F1–F5, F10, F11, and F14. The `cada_goto.smv` model focuses on control-flow transfers and defines memory-access predicates `prover_data_write`, `prover_data_read`, `mem_write_measured`, and `mem_read_measured`. It verifies F6–F9 and F12–F13.

For P1, `exec_state` tracks whether the prover or TCM is running, and `in_tcm_data` identifies whether the current address is in the TCM data region. F1 ensures that the prover cannot access TCM data. `dma_active` tracks DMA activity, and `DMA_APw` represents DMA write permission; F2 prevents prover DMA writes. The address variable `i` tracks memory accesses, while `APw` represents write permission. `utdm_b` and `utdm_e` bound TCM data, and F3 prevents the prover from writing evidence storage or key sets. `evidence_sent` and `sent_through_hook` enforce, through F4, that evidence can be sent only through the TCM hook. `utc_b` and `utc_e` bound the TCM code region, and F5 makes that region write-protected.

For P2, `in_trampoline` tracks trampoline-region execution, `APx` tracks execute permission, and `state` tracks execution state. F6 and F7 establish that trampoline execution is atomic and cannot be interrupted after it begins.

For P3, `prover_instrumented` tracks whether the prover is correctly instrumented, and `trampoline_call` identifies an invocation of the trampoline code. F8 permits such a call only from a correctly instrumented prover. `prover_executing` tracks prover execution, and F9 ensures that the prover runs only when instrumented.

For P4, the models use `in_prover_interval` to indicate that the program counter lies in the prover code region. `measure_now` is the conjunction of `ctrl_flow_measured`, `mem_write_measured`, and `mem_read_measured`. F10 requires every state in the prover code region to satisfy `measure_now`, enforcing continuous measurement during prover execution.

To obtain circumvention resilience, the system tracks the following critical events and requires immediate evidence recording:

- `ctrl_event`: a critical control-flow event in the prover code region, with evidence flag `ctrl_flow_measured`.
- `prover_data_write`: a write to the prover data region `adm`, with evidence flag `mem_write_measured`.
- `prover_data_read`: a read from the prover data region `adm`, with evidence flag `mem_read_measured`.

F11 verifies immediate recording after each critical control-flow event. F12 requires every write to the prover data region to be recorded without delay, while F13 applies the same rule to reads. Together, these formulas prevent critical events from bypassing the measurement process. F14 enforces multi-faceted measurement: whenever `measure_now` holds, all three evidence flags must be true, preventing a single evidence type from being accepted as a valid integrity measurement.

Formal verification in nuXmv presents several challenges. The 16-bit address space causes state explosion, addressed through concrete region boundaries such as `utc_b` and `utc_e` and through splitting verification into two models. DMA access is modeled using the enumerated type `dma_target_select` rather than unconstrained 16-bit values, enabling all critical memory regions to be checked without non-deterministic word assignments. Runtime atomicity is encoded precisely in LTL; F7 ensures that execution cannot terminate once the trampoline starts.

To maintain model fidelity, concrete regions—`utc`, `trampoline`, `utdm`, and `adm`—represent CADA's address space. A non-overlap constraint ensures that trampoline code and the hook function occupy distinct regions within the TCM code area `utc`. Specifically, the constraint `utc_b <= trampoline_b & trampoline_e < hook_b & hook_e <= utc_e` guarantees that trampoline code resides in `[trampoline_b, trampoline_e)` and the hook function resides in `[hook_b, hook_e)`, with no overlap. The access-control policies `APw` and `APx` enforce memory isolation and authorized execution. nuXmv confirms that all specified properties hold.
