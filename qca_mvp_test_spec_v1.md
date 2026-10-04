# QCA theory MVP: test specification v2

27 September 2026. Companion to: `defect_conditioned_code_selection_v1511.pdf` (v15), `acid_prophunt_hal_standing_connections_v1_annotated_review1.pdf`, `working_context_v4.1_addendum.md`, and the "Update Context" MVP note.

Purpose: a set of contained tests for an implementation agent. Each test states the question, the decision it gates, the procedure, the outputs and a decision rule. Do not expand scope beyond the stated procedure. Report numbers and configurations; interpret only where the decision rule asks for it.

Verification marks: [V] confirmed against the paper body in the 27 Sep session; [V-ref] confirmed via a citing paper's reference list; [unverified] do not rely on without checking.

---

## 0. Shared conventions

- **Source of truth for codes, graphs and schedules:** the ACID repository, github.com/riverlane/ACID. Do not re-derive graphs by hand where the repo builds them.
- **Naming trap:** ACID's "144-qubit" and "288-qubit" BB codes are *mid-cycle* codes with k = 12 whose dropout-free end-cycle distances are 6 and 12 [V, ACID §5]. Confirm which polynomials the repo uses for each and record them. Do not assume they are the [[144,12,12]] and [[288,12,18]] codes used directly as end-cycle codes.
- **Graph-construction check** (ACID App. D) [V]:

  | Instance | Qubits | Couplers |
  |---|---|---|
  | BB-144 hex | 144 | 432 |
  | BB-144 deg-5 | 144 | 360 |
  | BB-288 hex | 288 | 864 |
  | BB-288 deg-5 | 288 | 720 |
  | Surface d=11 square grid | 261 | 480 |
  | Surface d=11 hex grid | 241 | 340 |

  Fail loudly on any mismatch.
- **Reproduction noise model** (ACID App. C) [V]:
  - p = 0.001 depolarising after 1q and 2q gates;
  - bit or phase flips after reset and before measurement at rate p;
  - noiseless Pauli-product init and close;
  - R = dropout-free end-cycle distance;
  - decoder BP-AC.

  MVP noise-mapper tests use T8 instead.
- **Solver logging:** record for every solve:
  - OR-tools version;
  - `num_workers`, `random_seed`, time limit;
  - whether hints were supplied;
  - status (OPTIMAL / FEASIBLE / timeout);
  - final L.

  "K seeds" means varying `random_seed` with `num_workers = 1`. Also record one multi-worker run per configuration for comparison.
- **Statistics:**
  - Report LER per round with Wilson 95% CIs and the raw failure count.
  - Flag any LER estimated from fewer than 100 failures.
  - Compare distributions (quantiles), not only means.
- **Artifacts:** each test writes raw results (CSV or JSON), the exact config, and a README of at most 10 lines.

## 1. Dependency order

1. **T1 → T2 → T3 → T3b.** These decide whether heat maps can be built from orbit representatives, and whether the Anker–Debroy objective port is on the MVP critical path.
2. **T4, T5, T6:** independent and cheap. Run early.
3. **T7:** independent. It decides the secondary (asymmetric) testbed.
4. **T8, T9:** must finish before any MVP evaluation of HAL variants.
5. **T10, T11, T12:** parallel or later tracks.

---

## T1. Joint Automorphism Group and Orbit Census (BB family)

**Question.** Which single-coupler and single-qubit dropouts form equivalence classes under symmetries of the specified experiment, and how much compilation and simulation overhead can those symmetries eliminate?

**Gates.** The heat-map compute budget, eligibility for circuit transport in T2, and the potential resolution of single-defect logical-sensitivity weights. A small orbit count may limit this resolution, but it does not necessarily invalidate layout optimization.

**Inputs.**
- BB codes [[72,12,6]], [[90,8,10]], [[108,8,10]], [[144,12,12]], [[288,12,18]] from Bravyi et al., arXiv:2308.07915. Take (ℓ, m, A, B) from that paper's table, not from this document.
- The ACID repo's two BB instances, per the naming trap above.
- The degree-5 (Shaw–Terhal morphing) and degree-6 "hexagonal" connectivity variants.
Obtain code parameters from the cited source and confirm them against the repo instances using the shared naming and graph-construction checks. Mark combinations constructed outside the repository as agent-built and validate their construction before drawing symmetry conclusions.

**Experiment Pre-Checklist/Contract: Before Computing Performance Equivalences, Record the Following**
- Specify the hardware connections and the code (record-keeping stuff): The physical connectivity graph, stabilizer generators/stabilizer groups, chosen measured checks, and any qubit roles or gate restrictions.

- Specify what logical information we store and what counts as failure: Logical preparation, measured logical observables, and the exact encoded test failure event (individual logical error, basis-specific block failure, or another explicitly defined event).

- Specify how long we run and where errors occur: Memory duration, initialization & closing procedures, and the noise on each operation type. Return how many rounds R are performed.

- Specify how circuits and corrections are chosen: Compilation and schedule-selection policy, decoder settings, and permitted transformations of syndrome and logical labels.

- Specify the measured quantity plotted in the heat map: The fixed defect-free reference and heat map statistics. If you report the per-round LER, record its conversion from block failure and apply it consistently throughout the procedure.

**See if this statement holds for our experiment (Not to prove generally):** BB-288's arc-transitivity may arise from additional automorphisms, beyond translations, permitted by its 12x12 periodic structure but would be absent in BB-144’s 12x6 structure. Test whether these transformations exchange connection types that remain inequivalent in BB-144. Show the specific automorphisms that combine the translation-defined edge classes. Then, verify that they also connect all directed-edge classes. You should NOT infer arc-transitivity from the torus dimensions alone, but rather through explicit verification.

**Procedure.**
1. Build the global connectivity graph G and the stabiliser list with X/Z types. (Keep these distinct from a data-check Tanner graph unless they are explicitly the same graph in the experiment)
2. Compute three groups using nauty/pynauty or bliss (or an equivalent tool):
   - **(a)** The Graph Group: Aut(G) as the group of vertex permutations that preserve adjacency. This group should ignore which stabilizers we measure. A rotation could preserve the hardware connections while sending a check’s support to a set of qubits on which no chosen check exists.
   - **(b)** Typed-Check Group: The joint automorphism group that preserves G AND the chosen checklist, mapping X-stabiliser supports to X-stabiliser supports and Z to Z. That is, for an element of the group (b), a transformed X operator must be another check in the chosen X-check list. The analogous requirement holds for every Z-check. Individual checks may move or exchange labels; their types must remain the same.
   - **(c)** Global-exchange extension: Group b (Typed-Check Group), together with all valid transformations that allow a global XZ (and ZX) exchange. This must include a corresponding Pauli-basis transformation since a qubit permutation alone should not exchange X and Z. “Global” means that every X-check becomes a Z-check and every Z-check becomes an X-check. We are not allowing arbitrary partial exchanges.

  For (b), use a vertex-coloured auxiliary graph with separate colours for qubits, X-checks, Z-checks, and coupler subdivision nodes. Include required role labels. Coupler orbits are the orbits of subdivision nodes. For (c), explicitly test a globally type-swapped copy and, if an isomorphism exists, adjoin a verified exchange transformation. Do not obtain (c) merely by removing the X/Z colours: that can admit partial exchanges. Report orders of the induced physical transformations, excluding auxiliary-node permutations that act trivially on the physical system.

3. State what the checklist calculation captures. A symmetry can preserve the stabilizer group while mapping a listed generator to a product of generators, so group (b) may miss some code symmetries. When testing additional candidates, use the binary symplectic algebra (specified in the code's parameters) with phase tracking to verify that the transformed stabilizers generate the same group, including Pauli signs. You can only claim to have found all stabilizer-group automorphisms if completeness has been established (don't simply assume). These additional symmetries permit circuit reuse iff they also preserve or consistently transform the actual check-measurement protocol.

4. Identify a verified experiment-preserving subgroup $\Gamma_{\mathrm{exp}}$ of the candidates. Check the action on preparation, logical observables, failure event, noise, circuit conventions, and decoder. Note: an exchange relating X-memory and Z-memory is a relation between two experiments unless the defined metric and noise make it a symmetry of one experiment. Uniform gate-error rates alone do not establish experiment invariance. 

  To account for this, use an explicitly equivariant schedule policy (i.e., a scheduling whose compilation by nature respects relabeling). Transforming the defect and then compiling should produce the transformed version of the original circuit, including any required basis changes. Symmetry-based reuse requires both a consistent transformation of the experiment and a consistent choice of circuits. Independent finite-time solves are NOT assumed to be equivariant.

5. For each group ((a),(b),(c)) report:
   - the group order;
   - the number and sizes of qubit orbits;
   - the number and sizes of coupler orbits;
   - the number of orbits of unordered coupler pairs and of unordered qubit pairs (for 2-dropout sampling).

6. **Sanity check:** Check expected translations by explicitly checking the maps on the graph and typed checks. Where a faithful Z_ℓ × Z_m translation subgroup survives, we have that |(b)| ≥ ℓm. If it fails, search the construction or connectivity constraints for errors rather than imposing the bound unconditionally. Treat the original specification's BB-288 hex arc-transitivity and BB-144 hex multiple-edge-orbit statements (below) as reproduction targets to verify against the precise instances. Test directed- and undirected-edge transitivity separately.
**Validation against ACID §5** [V]:
   - BB-288 hex should be arc-transitive under (a).
   - BB-144 hex should have at least 2 edge orbits.
   - If (a) is arc-transitive but (b) is not, report it prominently. The group that licenses "same experiment" under uniform noise is (b), or (c) with the X/Z memory bases swapped.

7. Identify transformations that merge translation classes. Keep the proposed explanation involving the 12×12 versus 12×6 torus as a hypothesis until explicit maps establish it. Report prominently when graph symmetries fail type-check or experiment-level checks.

**Interpretation** For a truly invariant experiment and its performance statistic: $h(\sigma e)=h(e)$. Therefore, #{distinct heat-map values} $\leq|E/\Gamma_{exp}$

Generally, invariant functions have one free parameter per orbit, but values assigned to different orbits can coincide. An individual noisy estimate need not be constant within an orbit. Only if a verified subgroup is available does its orbits give a valid but potentially conservative reduction. (In simpler words, different symmetry classes can still happen to receive the same value; noisy data may temporarily violate the symmetry; and if you only trust some of the symmetries, you can still reduce the problem safely, just not as aggressively as if you knew the full symmetry group.)

**Output** Group/orbit tables; representative-to-member maps; the experiment pre-checklist; verified and rejected symmetry checks; completeness limitations; and projected compilation and simulation savings. Record runtime.

**Decision Rules** Permit representative-based reuse only for symmetries that pass T2 under the recorded experiment contract; uniform noise alone is a baseline and is insufficient to establish equivalence. Under these verified symmetries, the single-coupler heat map has one potentially independent value per coupler orbit, although different orbits may have equal values. 
If the MVP instance has three or fewer verified coupler orbits, flag “limited single-coupler weighting resolution” rather than declaring the heat map nearly uniform or uninformative: even two classes can provide useful layout guidance if their dropout penalties differ substantially. 
Use T3 to assess whether between-orbit differences are large enough, relative to their uncertainty, to support meaningful weighting. If they are not, use the T7 testbed or explicitly justify proceeding with another source of layout improvement. A uniform single-defect penalty limits the benefit of distinguishing couplers by isolated-dropout importance, but layouts can still differ through total defect exposure, operating noise, and multiple-defect effects.

---

## T2. Relabeling equivalence

**Question.** Is ACID's circuit for dropout {e} interchangeable, by qubit relabeling, with a circuit for {σ(e)} when σ is in group (b)?

**Procedure.**
1. Pick e and σ from T1 with σ(e) = e′ ≠ e.
2. Solve ACID for {e} with a fixed seed.
3. Apply σ to the emitted Stim circuit.
4. Build the relabeled circuit's detector error model. It must compile without non-deterministic detector errors.
5. Confirm the observables still span the logical space.
6. Simulate the original and the relabeled circuit under uniform noise.
7. Separately, solve {e′} from scratch with the same seed and simulate it.

**Output.** LER(e), LER(σ·circuit_e), LER(solve e′), each with CIs.

**Decision rule.**
- If LER(σ·circuit_e) agrees with LER(e) within CIs, one solve per orbit per seed is sufficient.
- This also holds under layout-dependent noise, because ACID does not see the layout: relabel, then simulate each relabeled circuit under its own per-edge noise.
- Differences between LER(e) and LER(solve e′) are solver variance by construction. Pass them to T3.

---

## T3. Solver-variance decomposition (BB and surface code)

**Question.** Is the within-bulk spread seen in ACID heat maps real location dependence, or solver variance?

**Gates.** Whether heat-map entries from single solves are interpretable.

**Procedure: BB.**
1. Use BB-144 hex (repeat on deg-5 if budget allows).
2. Choose one representative per coupler orbit from T1, plus 3 additional members of the largest orbit.
3. For each coupler, run K = 10 seeds at the ACID default time limit (record it).
4. For each seed, emit the circuit and simulate to at least 200 failures, or report the CI reached.

**Procedure: surface code.**
1. Use d = 11, square grid.
2. Choose 8 bulk couplers:
   - 4 pairs related by a symmetry of the patch (180° rotation; the 90° rotation composed with X↔Z where applicable);
   - 4 couplers at graded distance from the X and Z boundaries.
3. Run K = 10 seeds each, as for BB.
4. Include the dropout-free baseline with K = 10.

**Analysis.** Nested variance components on log(LER): seed within coupler, coupler within orbit (or symmetry pair), and between orbits. Report fractions and bootstrap CIs.

**Decision rules.**
- **Within-orbit variance ≈ 0 once seed variance is removed:** the orbit-representative method holds. Define each heat-map entry as the median over K seeds of the orbit representative.
- **Seed variance comparable to between-orbit differences:** single-solve heat maps are not interpretable. Run T3b before any HAL weighting.
- **Surface code:** any location dependence that survives seed averaging is genuine, because boundaries break symmetry. Do not attribute it to the solver.

**Cost.** This is the dominant cost. Budget it from T4.

## T3b. Variance reduction by anchoring (hints)

**Question.** Does warm-starting each dropout solve from a fixed canonical schedule cut seed variance without raising mean LER?

**Basis.** Anker–Debroy hint CP-SAT with the default LUCI schedule, which also guarantees an objective no worse than the default [V, arXiv:2512.10871 §IV C].

**Procedure.**
1. For the T3 couplers, pass as CP-SAT hints the dropout-free canonical schedule, restricted to the schedules that remain valid after the dropout.
2. Run K = 10 seeds.
3. Variant: pin the dropout-free baseline to the lowest-LER dropout-free solution found in T5.

**Output.** Mean and variance of log LER, hinted vs unhinted, per coupler.

**Decision rule.** If hinting at least halves the seed variance at no worse mean LER:
- adopt hinting as the MVP default;
- the Anker–Debroy objective port (T10) leaves the MVP critical path.

---

## T4. ACID runtime profile

**Procedure.**
1. Instances:
   - BB-144 hex and deg-5, and BB-288 hex;
   - 0, 1, 2, 3 coupler dropouts, and 0 or 1 qubit dropouts;
   - 5 random patterns each.
2. Record per solve:
   - shape-cache build time and model build time;
   - time to first feasible and time to best found;
   - time to proven optimum, or timeout;
   - final L and peak memory.
3. Record the machine spec.

**Output.** A cost model: seconds per solve by (instance, dropout count).

**Decision rule.** Sets the T3 and heat-map budgets. Compare against the "hours per heat map" figure in the Update Context note.

## T5. Dropout-free anomaly

**Question.** Does ACID's reported "a single dropped coupler usually lowers LER" on the 144-qubit hex instance [V, ACID §5] survive once the dropout-free baseline is not a single arbitrary solver draw?

**Procedure.**
1. Solve the dropout-free instance with K = 20 seeds. For each schedule record LER and the end-cycle code distances; ACID reports some schedules reach end-cycle distance 7 or 8 [V].
2. Solve single-coupler dropouts: one representative per orbit, K = 10 each.
3. Compare the full distributions.

**Decision rule.**
- If the best dropout-free seeds match or beat the single-dropout medians, the anomaly is a baseline-selection artifact. Pin the baseline (T3b variant) before any heat map feeds HAL.
- Define heat-map entries as ΔLER relative to the pinned baseline.
- Report negative entries; never clip them.

## T6. Anker–Debroy contributions other than the objective

**T6a. Gauge group (surface code).**
1. Reproduce the Anker–Debroy Fig. 4 configuration on d = 11 square grid: two perpendicular broken couplers at one qubit.
2. Confirm ACID emits a weight-1 quasi-stabiliser there.
3. Compute the dressed distance by gauge-fixing enumeration (ACID App. A). Compare against LUCI-style coarsening, where the coupler pair is converted into a broken qubit.

Expected from the source reading: ACID already has the "more complete" gauge group, because its quasi-stabilisers are connected components of the damaged local graph [V, ACID §3.1; Anker–Debroy §II].

**T6b. Weight-1 stabilisers (BB).**
1. On BB-144 hex and deg-5, draw 30 random patterns at each of 1, 2, 3 dropouts.
2. Count the weight-1 operators in ACID's output that lie in the stabiliser centre, as opposed to weight-1 gauge operators.

Expected: rare or absent, because weight-1 stabilisers arise at boundaries. This expectation is an inference, not a source claim.

**T6c. Excision (only if T6b finds any).** Implement excision per Anker–Debroy §III: remove qubits that support only a weight-1 stabiliser and are never a root or waypoint. Compare LER with and without it.

**Decision rule.** Record which of the three Anker–Debroy contributions apply to BB under ACID. Expected result: only the objective.

---

## T7. Asymmetric testbed code(s)

**Question.** Which qLDPC code with little symmetry can ACID compile today, so that the ACID → HAL heat map carries real information?

**Candidates, in priority order** (revised 27 Sep).
1. **Planar BB-derived codes:** Liang, Eberhardt & Chen, PRX Quantum 6, 040330 (arXiv:2504.08887). Weight-6, geometrically local, derived from BB codes, so the bulk is closest to ACID's BB shape cache.
   - Small test instance: (78, 6, 6).
   - Headline instances: (268, 8, 12) and (386, 12, 12).
2. **Tile codes:** Steffan et al., arXiv:2504.09171. Weight-6 [[288, 8, 12]]. HAL already includes tile-code layouts, on the Tanner graph, so re-run HAL on the ancilla-free graph.
3. **Radial codes:** Scruby, Hillmann & Roffe, arXiv:2406.14445. Only order-s rotational symmetry.
4. **Pruned BB codes** (Eberhardt, Pereira & Steffan, arXiv:2412.04181): fallback only.
   - The paper's explicit examples have k = 2 ([[30,2,4]], [[66,2,6]]).
   - A review notes that explicit stabiliser lists for the bivariate examples are not given.
   - It is kept on this list because ACID §5 names it [V], not because it is a competitive code.
5. **Surface code d = 5 or 7:** an ACID-side control only. It is planar, so HAL has nothing to do.

HAL reports that removing periodic boundaries significantly lowers hardware complexity [V, HAL abstract], so candidates 1 and 2 are also the hardware-relevant direction. Candidates 1 and 2 must each provide explicit stabiliser lists before use. Otherwise reconstruct the code from the paper and verify k and d.

**Connectivity construction.** No morphing circuit is required: ACID needs only that every check's local graph is connected [V, ACID §3].
- For each check, choose a Hamiltonian cycle on its support.
- In the bulk, reuse the BB hexagonal cycle.
- For truncated boundary checks, take the induced path of the bulk cycle plus one closing edge.
- Record the degree distribution and the edge count.

**Procedure.**
1. Build the code. Verify k, and estimate d with QDistRnd or a BP-OSD estimator.
2. Build the connectivity as above.
3. Run ACID dropout-free and record L, success and runtime. ACID records failure above L = 5 [V].
4. Run T1 on the instance.
5. Run 10 single-coupler dropouts split between bulk and boundary, K = 5 seeds each.

**Decision rule.** Adopt the first candidate that meets all three conditions as the MVP secondary testbed:
- dropout-free L ≤ 4;
- runtime within the T4-derived budget;
- at least 10 coupler orbits under group (b).

---

## T8. Noise mapper: specification and sanity

**Inputs.** A per-edge table from HAL routes:
- total routed length ℓ_e, in units of the shortest edge;
- tier sequence;
- bump transitions b_e;
- TSVs v_e.

HAL records route paths and switch-edge provenance. Build an adapter to a canonical table keyed by ACID edge IDs.

**Model (illustrative functional forms; every coefficient is swept, none asserted).**
- **Gate error:**
  `e_e = e_0 · (1 + α·(ℓ_e − 1)) · (1 + β_f · b_e)`
- **Coupler failure probability:**
  `p_e = 1 − (1 − p_b)^{b_e} · (1 − p_v)^{v_e} · (1 − p_ℓ · ℓ_e)`
- **Qubit failure:** i.i.d. p_q, independent of layout in v1.
- **Exclusion mode** (see T12): exclude a component whose sampled error rate exceeds τ.

**Sweep ranges.**
- α ∈ {0, 0.05, 0.1, 0.2}
- β_f ∈ {0, 0.01, 0.05, 0.1}
- p_b and p_v over two decades
- p_ℓ small

**Checks.**
- Monotonicity in every argument.
- Stock HAL and modified HAL are evaluated with the identical mapper and coefficients.
- Report per-edge distributions of e_e and p_e for each HAL variant.

**Note.** β_f ≈ 0 at b_e ≤ 2 is consistent with the flip-chip evidence cited in v15 §5.3. Behaviour at 5 to 10 transitions is unmeasured. That is the coefficient pair (fidelity and failure) to request from the HAL authors' group.

## T9. Statistical power budget

**Procedure.**
1. Take the baseline LER per round for the MVP instance at the candidate p, from T3 and T5.
2. Compute the failures needed per arm to resolve a relative effect δ ∈ {5%, 10%, 20%} at about 2σ:
   - unpaired: roughly 8/δ² failures per arm;
   - also compute the paired design, where both chips share the same dropout pattern.
3. Multiply by (dropout counts × patterns × HAL seeds × variants).
4. Convert to CPU-hours using the measured BP-AC decode rate.

**Decision rule.** Choose physical p, code size and pattern counts so the paired (E1) and unpaired (E2) comparisons resolve within budget. If they do not resolve at p = 1e-3 on the 144 instance, raise p or use a smaller instance, and document the choice.

For scale: Anker–Debroy's total gain on the surface code was 14.5% / 23.6% [V].

---

## T10. Anker–Debroy objective port (parallel track)

This is not on the MVP critical path unless T3/T3b say it is.

**T10a. Acceptance test.** On the dropout-free surface code, minimising the ported objective must return one of the four symmetric canonical syndrome circuits [V, Anker–Debroy §V].

**T10b. Term-mapping audit.** For each term, write its definition in ACID's IR (quasi-stabiliser, rooted spanning tree, timing):

| Term | Symbol | Weight |
|---|---|---|
| Deterministic measurements | m | −1 (normalising) |
| Skip twice | s2 | 6 |
| Skip thrice | s3 | 5 |
| Alignment | a | 12 |
| Basis changes | b | 2 |

Weights are [V, Anker–Debroy Eq. 4]. Flag any term whose LUCI definition depends on shape geometry; "stretches", inside alignment, is the known hard case.

**T10c. Figure-13 analogue.** On BB-144 hex single dropouts, compute the Spearman correlation between objective value and LER across time-limited solves.

**Decision rule.** Port only terms that pass T10b. Report the failed terms and why.

## T11. Crosstalk: propagation weight and co-activity map

**T11a. Propagation weight in ancilla-free circuits.**

Di Bella's Theorem 2 (a single pair event propagates to a weight-2 data Pauli) is proved only for the depth-8 ancilla-based BB schedule [V, arXiv:2604.01040].

Procedure:
1. In an ACID circuit, find every pair of CNOTs active in the same timestep.
2. Inject the correlated pair Pauli on the pair.
3. Propagate it through the remainder of the contraction layer with Stim.
4. Record the resulting data-error weight distribution.

If weights above 2 occur, the retained single/pair truncation must be re-derived before the exposure metric is used.

**T11b. Co-activity map.** For each unordered coupler pair (e, e′), compute three versions:
- **(i)** the tick co-activity count from the dropout-free schedule (the set A_t in Di Bella's notation);
- **(ii)** the same, averaged over sampled dropout patterns using the layout-independent prior;
- **(iii)** a logical-weighted version: count a co-activity only if the propagated pair error from T11a lands inside a sampled minimum-weight logical support.

Report the Spearman correlations between (i), (ii) and (iii).

Decision rules:
- If ρ((i), (ii)) > 0.9, use the dropout-free map.
- If ρ((i), (iii)) is low, raw counts are the wrong quantity. Precedent: under Di Bella's logical-aware layout the crossing count rose from 522 to 542 while exposure and LER fell [V].

**T11c. Reproduce before extending.**
1. Reproduce Di Bella's BB72 reference point from the released code (Zenodo 10.5281/zenodo.19337541; github.com/angelodibella/works). Target: monomial LER 0.2868 vs biplanar 0.0581 at (p, α, J0τ) = (1e-3, 3, 0.04) [V].
2. Only after reproduction, attach the kernel to HAL geometry, using closest-approach separation that includes tier spacing.

## T12. Exclusion threshold (later)

**Basis.** Drouet et al., arXiv:2607.12118 [V].
- ibm_miami median 2q error 2.83e-3, worst coupler 1.69e-1.
- Readout median 2e-2, worst 0.15.
- Exclusion thresholds used:
  - couplers above 5% error or without calibration data;
  - measure qubits above 7e-2 readout error;
  - data qubits with both T1 and T2 below 80 µs.

**Procedure.**
1. Sample per-component error rates from a heavy-tailed distribution anchored to these figures.
2. Sweep τ.
3. For each τ: exclusion set → ACID → LER.
4. Compare against (a) a dead-only model and (b) no exclusion with noise-informed decoding.

**Decision rule.** Determines whether the MVP's dead/alive dropout model understates layout effects.

---

## Out of scope

PropHunt rework, multi-code families, decoder provisioning, porting the full Anker–Debroy ILP to LUCI IR.
