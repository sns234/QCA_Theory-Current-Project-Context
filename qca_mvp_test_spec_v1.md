# QCA theory MVP: test specification v2.2

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

## T1: Joint Automorphism Group and Orbit Census (BB family)

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
## T2: Relabeling equivalence

**Question**: Given a verified symmetry $\sigma \in \Gamma_{\mathrm{exp}}$ from T1 that maps a dropped coupler $e$ to $e'=\sigma(e)$, can an ACID circuit compiled for ${e}$ be transported by qubit relabeling to ${e'}$ without changing the defined memory experiment or its logical error rate?

**Gates**: Whether one compilation per coupler orbit per solver seed is sufficient to generate the circuits used for the single-dropout heat map.

**Important distinction**: Two different claims must be tested separately:

1. Circuit transport equivalence: A circuit compiled for $e$ remains valid when transformed by $\sigma$ to the dropout location $e'$, including the corresponding transformations of detectors, logical observables, and measurement conventions.

2. Independent compilation equivalence: Running ACID independently on $e$ and $e'$ produces circuits with equivalent logical performance.

The first claim follows from a properly verified symmetry and consistent circuit transport. The second does not follow automatically, since ACID's finite-time optimization may select different schedules even for symmetry-related dropouts. We want to explore this difference within our project.

**Inputs**:

1. The global connectivity graph, typed stabilizers, and experiment contract from T1.

2. The experiment-preserving subgroup $\Gamma_{\mathrm{exp}}$, including explicit vertex permutations.

3. The ACID compilation settings and uniform reproduction noise model defined in Section 0.

4. The memory duration $R$, decoder configuration, and precise logical failure event.

**Procedure**:

1. Select one representative coupler $e$ from each nontrivial coupler orbit of $\Gamma_{\mathrm{exp}}$. For each representative, choose a verified transformation $\sigma$ such that $\sigma(e)=e'\ne e$. Record both the permutation and the orbit mapping.

2. Compile an ACID circuit $C_e$ for dropout ${e}$ using a fixed solver seed and time limit. Save the complete Stim circuit, the selected contraction schedule, and solver metadata.

3. Construct the transported circuit $\sigma(C_e)$ by applying the physical-qubit permutation to every operation. Transform any other labels or operations required by the experiment contract, including logical-observable conventions and Pauli-basis changes if applicable.

4. Verify structural validity.

5. Every two-qubit operation must act on an available coupler after the dropout has been transformed.

6. The transformed circuit must preserve the intended stabilizer-measurement protocol.

7. Its Stim detector error model must compile without unexpected nondeterministic detectors.

8. The logical observables must remain a complete and independent representation of the intended logical information.

9. Detector and observable relationships must agree with those of the original circuit under the recorded transformations.

10. Verify that the noise model and decoder transform consistently. Under uniform physical noise, the transported experiment must have the same noise-channel structure after relabeling. Check that the decoder and the definition of logical failure do not introduce a preferred labeling.

11. Simulate $C_e$ and $\sigma(C_e)$ under the same uniform noise model. Report the logical error rates per round, raw failure counts, and Wilson 95% confidence intervals. Also report a confidence interval for their difference. Choose and record an equivalence tolerance before inspecting the results; overlapping individual confidence intervals alone are not sufficient to establish equivalence.

12. Independently compile a circuit $C_{e'}$ for dropout ${e'}$, using the same nominal solver seed, time limit, and settings. Simulate this circuit and compare its LER against the transported circuit. Do not assume the identical seed produces symmetry-related schedules, since solver variable ordering and finite-time search may differ.

**As a separate sanity check**: simulate the transported circuit using nonuniform, layout-dependent per-edge noise. Each circuit must use the noise assigned to its own physical edges. No equality of logical error rates is expected unless the physical noise assignment also respects the symmetry.

**Output**:

1. The tested permutations, corresponding orbit representatives, and representative-to-member maps.

2. Original, transported, and independently compiled Stim circuits.

3. Results of the connectivity, detector, and logical-observable checks.

4. LER $(C_e)$, LER $(\sigma(C_e))$, and LER $(C_{e'})$, with raw failure counts and confidence intervals.

5. Differences between the three measured error rates, with uncertainty estimates.

6. Solver settings, circuit depths, and runtimes.

7. A list of accepted and rejected circuit-transport symmetries.

8. Decision rules.

**Transport passes**: If the structural checks pass and the logical-performance difference is statistically consistent with the prespecified equivalence tolerance, permit representative-based circuit reuse for the tested experiment and symmetry. A statistically inconclusive comparison is not a pass.

**Transport fails**: If the circuit is invalid, the logical observables do not transform correctly, or the decoder/noise implementation breaks the claimed equivalence, reject that transformation for representative-based reuse until the failure is understood.

**Independent compilations differ**: Differences between $C_{e'}$ and $\sigma(C_e)$ do not automatically invalidate circuit transport. They instead demonstrate sensitivity to the compilation or schedule-selection policy and must be investigated in T3.

**Layout-dependent noise**: Transported circuits may still be reused as valid circuits under different edge-dependent noise assignments, but their LERs must be simulated separately. Circuit reuse does not imply equal performance or optimality under those assignments. This gets back to the point we identified earlier in the section where we discussed the two conditions we wanted to test: circuit transport vs. independent compilation equivalences. 

Permit one compilation per orbit per seed only for transformations satisfying the required structural and experiment-level checks. The heat map must use the same method for choosing a circuit at every dropout location. For example, if we select the best-performing circuit from 10 solver seeds for one representative coupler, we must use that same selection rule for every other representative. Otherwise, differences in logical error rates could reflect how circuits were selected rather than the actual effects of the dropouts.

## T3: Solver-variance decomposition (BB and surface code)

**Question**: To what extent does the observed variation in single-coupler dropout heat maps reflect the physical location of the dropout, rather than ACID's choice of contraction schedule or finite-sample simulation noise?

**Gates**: Whether heat maps produced from single solver runs contain meaningful spatial information, and whether multiple seeds or an anchored compilation policy are required before assigning logical-sensitivity weights to hardware connections.

**Motivation**: A single dropout location can admit multiple valid syndrome-extraction schedules. ACID optimizes circuit scheduling using a heuristic objective rather than directly minimizing logical error rate. Consequently, different solver seeds may return circuits with different logical performance, even when the code, connectivity, and dropout pattern are unchanged.

For couplers related by a verified experiment symmetry, the underlying physical experiment is equivalent under consistent circuit transport. Nevertheless, independently compiled schedules may exhibit different logical error rates.

Thus, the variation in an observed heat map should be decomposed into:

1. Simulation uncertainty: Statistical variation from the finite number of sampled logical failures.

2. Solver-seed variation: Different schedules obtained for the same dropout location.

3. Within-orbit variation: Differences between independently compiled circuits for dropout locations related by verified experiment symmetries.

4. Between-orbit variation: Differences between coupler classes not related by the verified symmetry group.

The last component is the principal candidate for symmetry-resolved logical-sensitivity information, but must be large enough relative to the other sources of uncertainty to be useful.

**Procedure A:** Bivariate bicycle codes

Use BB-144 with the degree-6 hexagonal connectivity. Repeat on the degree-5 connectivity if permitted by the T4 runtime budget, and from T1, select:

1. One representative coupler from each verified coupler orbit of $\Gamma_{\mathrm{exp}}$.

2. Three additional members of the largest nontrivial coupler orbit.

If T2 rejects representative-based transport for a candidate symmetry, retain the candidate orbit labels for exploratory comparison but do not assume equivalent performance.

For each selected coupler, compile $K=10$ independent ACID schedules by varying random_seed, holding num_workers = 1 and the solver time limit fixed. Use the same set of ten nominal seed values across all locations for reproducibility. Record the solver status and circuit depth for every run. Simulate every generated circuit using the shared uniform reproduction noise model. Target at least 200 logical failures per circuit; if you can't reach this within the allocated budget, report the attained failure count and confidence interval. Flag estimates with fewer than 100 failures. Ensure that we include a dropout-free reference. Use the multi-seed dropout-free results of T5 when available, rather than selecting an arbitrary single-solve baseline.

For symmetry-related locations, compare both independently compiled circuits at each location and circuits transported from the common orbit representative using the T2 procedure. The transported comparison acts as a control for determining whether observed within-orbit differences originate from independent scheduling rather than physical inequivalence.

**Procedure B**: Surface code

Use the distance-11 surface code on the square-grid connectivity.

Select 12 couplers in total:
1. Four symmetry-related pairs (eight couplers). Include pairs related by 180° rotation and, where valid, transformations involving 90° rotation and global $X\leftrightarrow Z$ exchange.
2. Four additional couplers at graded distances from the X and Z boundaries, to probe genuine boundary-dependent effects.

For transformations exchanging X and Z, ensure that we verify whether they preserve the chosen memory experiment or instead relate an X-memory experiment to a Z-memory experiment. Do not combine the latter measurements without recording and accounting for the change of logical basis.

Compile $K=10$ independently seeded circuits per selected coupler, using the same settings as Procedure A. Simulate each circuit to the same statistical target as in Procedure A. Then, compile and simulate the dropout-free surface-code instance with $K=10$ seeds.

**Analysis:**

1. Report the LER per round for every circuit, including its raw failure count and Wilson 95% confidence interval.

2. Estimate the variation in log(LER) using a nested model with:

3. Solver seed within dropout location.

4. Dropout location within verified orbit or symmetry class.

5. Variation between orbit or location classes.

6. Separate finite-shot Monte Carlo uncertainty from solver-induced variation. Use an appropriate binomial-error model or explicitly documented regularization when constructing log(LER) estimates, so that zero observed failures do not produce undefined values.

7. Report estimated variance components, their relative contributions, and bootstrap confidence intervals. Compare full distributions and quantiles rather than relying only on means.

8. Compare the magnitude of between-orbit differences against both solver-seed variance and simulation uncertainty. Report the uncertainty in the estimated difference between orbit-level LERs.

For symmetry-related couplers, compare independently compiled results against the circuit-transport controls from T2. A reproducible difference between independently compiled schedules does not, by itself, demonstrate intrinsic location dependence. For the surface code, determine whether systematic differences among inequivalent locations survive seed averaging. Interpret these differences relative to boundary geometry and the chosen scheduling policy.

**Output**

1. Per-location and per-seed LER data, with confidence intervals and failure counts.
2. Solver objective/status, circuit depths, and selected schedules.
3. Within-location, within-orbit, and between-orbit variance estimates.
4. Bootstrap uncertainty estimates for the variance components.
5. Comparisons between independently compiled and transported circuits.
6. Distributions of logical-sensitivity penalties relative to the dropout-free baseline.

**Decision rules**

1. Small seed variance and meaningful between-orbit differences: Representative-based heat maps are suitable for assigning orbit-level logical-sensitivity weights, provided T2 has validated the relevant symmetry transport.
2. Seed variance comparable to or larger than between-orbit variation: A heat map from a single ACID solve is not reliable for assigning relative importance to individual couplers. Run T3b before using it in HAL.
3. Within-orbit differences disappear under circuit transport: Treat differences between independent solves as effects of schedule selection, not physical location dependence.
4. Between-orbit differences are unresolved: Flag insufficient single-coupler weighting resolution. Consider the asymmetric T7 testbed or explicitly justify another source of layout improvement.
5. **Surface-code boundary effects**: Differences that persist after seed averaging are reproducible under the tested compilation policy, but must not automatically be attributed entirely to physical location if independently selected schedules remain a confounding factor.
6. Once the compilation policy and baseline are finalized, define the heat-map entry using a consistent multi-seed summary (the median over $K$ seeds by default) and the pinned dropout-free reference determined in T5. Preserve negative dropout penalties rather than clipping them.

**Cost** We expect T3 to dominate the compilation and simulation budget. Use the measured runtime distribution from T4 and the decoding throughput from T9 to estimate its total cost before execution.

## T3b. Variance reduction by anchoring (solver hints)

**Question**: Can warm-starting ACID from a fixed dropout-free reference schedule reduce solver-seed variance without worsening average logical error rate?

**Gates**: Whether warm-started scheduling should become the MVP default, and whether the Anker–Debroy objective port in T10 is needed on the MVP critical path.

**Basis**: Anker–Debroy use CP-SAT hints based on a canonical LUCI schedule to improve optimization stability and performance [V, arXiv:2512.10871 §IV C]. The effectiveness of the same approach inside ACID must be measured independently.

**Procedure**

1. Select a fixed dropout-free reference schedule for each test instance. Record its contraction assignments, depth, solver configuration, and logical performance.
2. For every dropout location sampled in T3, identify assignments from the reference schedule that remain valid following the dropout. Use these assignments as CP-SAT hints. Record which hints are retained or discarded.
3. Run $K=10$ independent solver seeds for each dropout pattern with hints enabled, using the same time limits and noise model as the unhinted T3 experiments.
4. Compare the hinted and unhinted schedules using the mean and variance of log(LER), median LER/other relevant quantiles, final circuit depth and solver status, and time to first feasible solution and final solution quality.
6. Repeat using the lowest-LER dropout-free reference found in T5 as an alternative anchoring schedule.
7. Check whether the hints produce a feasible incumbent and whether they alter the final schedules. Supplying partial CP-SAT hints does not by itself guarantee an improved circuit or LER.

**Output**: Per-coupler distributions of logical error rate, variance, circuit depth, and solver runtime for hinted and unhinted compilation.

**Decision rules**: If hinting reduces solver-seed variance by at least 50% without a statistically supported degradation in mean LER, adopt it as the default MVP compilation policy. If hinting succeeds, leave the Anker–Debroy objective port (T10) off the MVP critical path. If hinting fails to sufficiently reduce variance, retain the unhinted results, record the failure mode, and assess whether T10 or increased seed averaging is required before constructing the HAL sensitivity weights.
<!--## T3. Solver-variance decomposition (BB and surface code)

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
-->
---
## T4. ACID runtime profile

Question. What are the measured compilation costs of ACID across the proposed BB-code instances and dropout patterns, and how much computation will be required for T2, T3, T5, and the eventual sensitivity heat maps?

Gates. The practical compilation budget, number of feasible independent solver runs, and viability of the orbit-representative strategy.

Procedure.

Profile the following ACID configurations:

BB-144, degree-6 hexagonal connectivity.

BB-144, degree-5 connectivity.

BB-288, degree-6 hexagonal connectivity.

For each configuration, test the Cartesian product of:

Coupler dropout count: $n_e\in{0,1,2,3}$.

Qubit dropout count: $n_q\in{0,1}$.

Generate five distinct random dropout patterns for each nonzero dropout-count combination. Use one dropout-free baseline per instance. Sample patterns with a recorded random seed, and avoid counting a coupler incident to an already dropped qubit as an additional independent dropout.

Use the repository's native graph and code constructors, enforcing the graph-construction checks in Section 0 before profiling.

For every configuration and sampled pattern, record:

Shape-cache construction time.

Scheduling/model construction time.

Time to first feasible solution.

Time to best solution found.

Time to proven optimum, if applicable.

Total wall-clock compilation time.

Final contraction-layer count $L$.

Solver status and termination reason.

Peak memory usage.

Record the solver configuration for every run:

OR-Tools version.

num_workers.

random_seed.

Time limit.

Whether hints were supplied.

Any relevant ACID configuration or objective settings.

Profile the standard single-worker setting used for controlled seed comparisons. Include one multi-worker run per configuration to identify the effect of solver parallelism. Do not interpret multi-worker runs as independent single-worker seeds.

Distinguish cold-start and warm-cache timings. If shape information is reused across patterns, report the cost of constructing it separately from the incremental cost of each additional solve.

Record the machine specifications, including CPU, core count, RAM, software versions, and whether runs were executed under shared computational load.

Analysis

Group compilation times by code instance, connectivity type, coupler dropout count, and qubit dropout count.

Report the median, 90th percentile, maximum, and observed timeout rate for each configuration.

Separate the costs of graph and shape preprocessing, solver model construction, and optimization.

Identify how runtime changes with increasing numbers of defective components. Distinguish longer optimization times from failures to find a feasible schedule.

Use the observed runtime distributions to estimate the compilation cost of:

T2 circuit-transport validation.

T3 multi-seed variance experiments.

T5 dropout-free baseline studies.

Full single-coupler heat maps, with and without verified orbit-based circuit reuse.

Report compilation costs separately from logical-error simulation costs. The latter will be combined with the BP-AC decoding throughput measured for T9.

Output.

Raw per-solve timing and solver-configuration data.

Runtime distributions by code instance and dropout configuration.

Cold-cache versus warm-cache overhead.

Solver success, timeout, and depth statistics.

Estimated CPU-hours for the principal MVP experiments.

Projected compilation savings from the verified orbit-representative strategy.

Decision rules.

Use the measured runtime distributions, rather than the runtime of a single representative solve, to establish the computational budget for T3 and the complete heat maps.

Compare projected costs against the "hours per heat map" estimate in the Update Context MVP note.

If the projected cost exceeds the available budget, report which component dominates and how much representative-based reuse or reduced solver variance would save. Do not silently reduce the prescribed sample sizes.

Use the results to determine whether the BB-288 instance is practical for the full MVP study or should be limited to validation experiments.
<!--
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
-->
## T5. Dropout-free anomaly

**Question.** ACID reports that removing a single coupler from BB-144 with hexagonal connectivity often *lowers* the logical error rate (LER). Is this a genuine consequence of the dropout, or does it occur because the dropout-free solver selects a suboptimal syndrome-extraction schedule?

**Gates.** Whether the dropout-free reference used to calculate the heat map is representative, and whether its schedule must be fixed before evaluating dropout penalties.

**Procedure.**

1. Compile the dropout-free BB-144 hex instance using $K=20$ independent solver seeds. Record each circuit's LER, contraction-layer count $L$, and end-cycle code distance. ACID reports that some schedules achieve end-cycle distances of 7 or 8 [V, ACID §5].
2. For each coupler orbit identified in T1, select one representative and compile $K=10$ independently seeded single-dropout circuits.
3. Simulate all circuits using the same noise model and memory experiment, reporting LER distributions and confidence intervals.
4. Compare the best-performing dropout-free schedules against the single-dropout distributions. Validate any selected baseline using independent simulation samples to avoid selecting a baseline solely because of statistical fluctuations.

**Output.** Dropout-free and single-dropout LER distributions, associated end-cycle distances, solver configurations, and a proposed fixed reference circuit.

**Decision rules.**

- If the best dropout-free schedules match or outperform the single-dropout medians, treat baseline selection as a plausible explanation for the anomaly and adopt a validated dropout-free reference.
- Define each heat-map entry as $\Delta\mathrm{LER}(e)=\mathrm{LER}(e)-\mathrm{LER}_0$, where $\mathrm{LER}_0$ is the fixed dropout-free reference.
- Preserve negative values of $\Delta\mathrm{LER}$ rather than setting them to zero. A negative value indicates that the tested dropout circuit performed better than the reference, not necessarily that the physical defect is beneficial.

---

## T6. Anker–Debroy contributions other than the objective

**Question.** Which improvements proposed by Anker–Debroy are already supported by ACID, and which would require additional implementation?

**Gates.** Whether ACID requires changes to its subsystem-code construction or qubit-excision procedure before use in the MVP.

### T6a. Gauge-group construction (surface code)

1. Reproduce the Anker–Debroy Fig. 4 configuration on a distance-11 square-grid surface code, with two perpendicular broken couplers incident on one qubit.
2. Check whether ACID produces the expected weight-1 quasi-stabilizer.
3. Calculate the resulting dressed code distance using the gauge-fixing procedure in ACID Appendix A.
4. Compare against LUCI-style coarsening, where the same pair of broken couplers is treated as a defective qubit.

**Expected result.** ACID already constructs the more complete gauge group by splitting damaged local stabilizer graphs into connected components [V, ACID §3.1; Anker–Debroy §II].

### T6b. Weight-1 stabilizers (BB codes)

1. For BB-144 hex and degree-5 connectivities, sample 30 random dropout patterns for each dropout count of 1, 2, and 3.
2. Identify all weight-1 operators produced by ACID.
3. Distinguish weight-1 operators belonging to the stabilizer center from those belonging only to the gauge group.

**Expected result.** Weight-1 stabilizers should be rare or absent for periodic BB codes. This is a hypothesis to test, not an established result.

### T6c. Qubit excision (conditional)

If T6b identifies weight-1 stabilizers, implement the excision procedure of Anker–Debroy §III for eligible qubits that are not required as contraction roots or waypoints. Compare logical error rates before and after excision under otherwise identical conditions.

**Output.** Gauge-group and dressed-distance comparisons, weight-1 stabilizer counts, and excision results where applicable.

**Decision rule.** Determine which Anker–Debroy improvements are absent from ACID. The working expectation is that the main remaining improvement for BB codes is the optimization objective investigated separately in T10.

---

## T7. Asymmetric testbed codes

**Question.** Which qLDPC code with relatively little symmetry can ACID compile efficiently, allowing the resulting single-dropout heat map to distinguish a meaningful number of physically inequivalent couplers?

**Gates.** Selection of a secondary MVP testbed if the highly symmetric BB instances provide insufficient logical-sensitivity resolution.

**Candidates (priority order).**

1. **Planar BB-derived codes:** Liang, Eberhardt & Chen, arXiv:2504.08887. Begin with $[[78,6,6]]$; larger candidates include $[[268,8,12]]$ and $[[386,12,12]]$. Their BB-derived structure may be compatible with ACID's existing contraction-schedule methods.
2. **Tile codes:** Steffan et al., arXiv:2504.09171. Consider the weight-6 $[[288,8,12]]$ code. HAL already supports tile-code layouts, but these must be regenerated using the ancilla-free connectivity graph required by ACID.
3. **Radial codes:** Scruby, Hillmann & Roffe, arXiv:2406.14445. Their limited rotational symmetry may offer greater spatial sensitivity.
4. **Pruned BB codes:** Eberhardt, Pereira & Steffan, arXiv:2412.04181. Fallback candidates, including $[[30,2,4]]$ and $[[66,2,6]]$.
5. **Surface code ($d=5$ or $7$):** Use only as an ACID-side control, since its planar connectivity offers little reason for HAL optimization.

**Connectivity construction.** ACID requires a connected local graph for each measured check, rather than a pre-existing morphing circuit. Construct a Hamiltonian cycle over each weight-6 check where feasible, following the BB hexagonal construction. For truncated boundary checks, use the induced path and a closing edge. Record the resulting physical connectivity, vertex degrees, and coupler count.

**Procedure.**

1. Construct each candidate code from verified stabilizer generators. Confirm $k$ and estimate or verify $d$ using QDistRnd or an appropriate distance estimator.
2. Construct and validate the global connectivity graph, confirming that every check's local connectivity is connected.
3. Attempt dropout-free ACID compilation. Record the minimum feasible contraction-layer count $L$ and runtime.
4. Perform the T1 automorphism census, including the typed-check group (b).
5. For viable candidates, test 10 single-coupler dropouts distributed across bulk and boundary locations, with $K=5$ solver seeds per location.

**Output.** Verified code parameters, graph statistics, compilation runtimes, coupler-orbit counts, and preliminary LER distributions.

**Decision rule.** Adopt the first candidate satisfying all three conditions:

- Dropout-free circuit with $L\leq4$.
- Compilation runtime within the T4-derived budget.
- At least 10 coupler orbits under the typed-check automorphism group (b).

This establishes the secondary testbed; it does not by itself prove that the orbit-dependent LER differences will be large enough to benefit HAL.

---

## T8. Noise mapper: specification and sanity

**Question.** How should the physical geometry of each HAL-routed coupler be translated into gate errors and permanent-failure probabilities, so that different layouts can be compared under a consistent hardware model?

**Gates.** A common noise model for evaluating stock HAL against defect-aware HAL.

**Inputs.** For every physical coupler $e$, obtain from HAL:

- Normalized routed length $\ell_e$, measured relative to the shortest coupler.
- Routing-tier sequence.
- Number of bump transitions $b_e$.
- Number of TSV transitions $v_e$.

Create an adapter that maps HAL's routed edges onto the same canonical physical-coupler IDs used by ACID. Verify that both programs operate on the same ancilla-free connectivity graph.

**Noise model.** Use the following illustrative parameterization, sweeping the coefficients rather than treating them as experimentally established constants.

1. **Two-qubit gate error:**

   $$
   e_e=e_0[1+\alpha(\ell_e-1)](1+\beta_f b_e).
   $$

   Here, $e_0$ is the reference gate-error probability, $\alpha$ captures sensitivity to coupler length, and $\beta_f$ captures additional error associated with bump transitions.

2. **Permanent coupler failure probability:**

   $$
   p_e=1-(1-p_b)^{b_e}(1-p_v)^{v_e}(1-p_\ell\ell_e).
   $$

   Here, $p_b$, $p_v$, and $p_\ell$ represent contributions from bumps, TSVs, and routed length, respectively. Restrict sampled parameters so all resulting probabilities remain valid.

3. **Qubit failures:** Model each qubit as failing independently with probability $p_q$, independent of layout in this first version.

**Parameter sweeps.**

- $\alpha\in\{0,0.05,0.1,0.2\}$.
- $\beta_f\in\{0,0.01,0.05,0.1\}$.
- Sweep $p_b$ and $p_v$ over two decades.
- Choose small values of $p_\ell$ consistent with valid failure probabilities.

**Procedure.**

1. Extract the per-coupler geometry table from each HAL layout and verify the ACID edge mapping.
2. Apply identical noise formulas and parameter values to stock HAL and modified HAL.
3. Verify that predicted errors and failure probabilities are nondecreasing with routed length, bump count, and TSV count over the permitted parameter domain.
4. Compare the resulting per-edge distributions of $e_e$ and $p_e$ across layouts.
5. Export these distributions for the dropout sampling and circuit-level simulations used in the MVP evaluation.

**Output.** A canonical edge-to-noise table for each layout, parameter-sweep configurations, and distributions of gate-error and coupler-failure probabilities.

**Decision rules.**

- Accept the mapper only if it passes probability-range, monotonicity, and edge-consistency checks.
- Evaluate all HAL variants using the same noise model and coefficients.
- Treat conclusions that depend strongly on uncertain coefficients as sensitivity results, not established hardware predictions.

**Hardware caveat.** The flip-chip evidence discussed in v15 §5.3 suggests a small bump-induced fidelity penalty at low bump counts, but behavior at 5–10 transitions remains insufficiently characterized. The dependence of gate fidelity and coupler failure on repeated bump transitions should be identified as a quantity requiring additional hardware measurements.

## T9. Statistical power budget

**Question.** How many logical-error samples and simulation hours are required to reliably distinguish the performance of stock HAL from defect-aware HAL?

**Gates.** Whether the proposed MVP experiments can resolve meaningful improvements in logical error rate within the available computational budget.

**Motivation.** Logical failures may be rare, so a small apparent improvement in LER could result from statistical fluctuations rather than a genuine advantage of the modified layout. We must therefore determine the required sample size before conducting large-scale simulations.

**Procedure.**

1. Obtain the baseline LER per syndrome-extraction round from T3 and T5, using the selected physical error rate $p$ and fixed reference circuit.

2. Estimate the number of logical failures required to distinguish relative LER improvements of

   $$
   \delta\in\{0.05,0.10,0.20\}
   $$

   at approximately $2\sigma$ statistical significance.

   For two independent samples with similar, sufficiently small LERs, use the approximate requirement

   $$
   N_{\mathrm{fail}}\approx\frac{8}{\delta^2}
   $$

   failures per comparison arm.

3. Calculate the corresponding sampling requirements for two experimental designs:
   - **E1 (paired):** Compare both HAL variants using matched dropout patterns. Estimate the uncertainty of the paired LER difference, accounting for correlations between the two outcomes.
   - **E2 (unpaired):** Sample independently fabricated chips for each HAL variant, using each layout's own component-failure probabilities.

4. Estimate the total simulation workload by multiplying the required samples across dropout counts, sampled patterns, HAL random seeds, and layout variants. Account for the number of circuits simulated per configuration.

5. Measure the BP-AC decoder throughput and convert the projected number of simulated shots into CPU-hours. Include the cost of any required ACID recompilation.

6. Compare the projected computational cost against the available budget. If necessary, evaluate different code sizes or physical error rates while keeping the comparison method fixed.

**Output.** Required failure counts, simulation-shot estimates, expected CPU-hours, and detectable relative LER improvements for both experimental designs.

**Decision rules.**

- Select the code size, physical error rate, and number of dropout patterns such that the intended improvement can be resolved within the available budget.
- If a 5% improvement is too expensive to detect, determine whether 10% or 20% improvements are statistically accessible.
- If the BB-144 instance at $p=10^{-3}$ is prohibitively expensive, consider increasing $p$ or switching to a smaller instance, documenting the effect on the experiment.
- Do not claim an improvement when the observed difference is smaller than its statistical uncertainty.

**Reference.** Anker–Debroy report improvements of approximately 14.5% and 23.6% for their surface-code benchmarks [V]. These provide useful scales for selecting detectable effect sizes, but are not predictions of HAL's performance.

---

## T10. Anker–Debroy objective port (parallel track)

**Question.** Can the objective function introduced by Anker–Debroy for optimizing LUCI syndrome-extraction schedules be adapted to ACID to improve the logical performance of its compiled circuits?

**Gates.** Whether modifying ACID's optimization objective provides a meaningful performance advantage over its existing scheduling heuristic.

**Priority.** This is a parallel investigation and does not belong on the MVP critical path unless T3 and T3b indicate that solver-seed variance or poor schedule selection prevents reliable heat-map construction.

### T10a. Acceptance test (surface code)

1. Implement the Anker–Debroy objective within ACID's CP-SAT formulation, preserving ACID's existing scheduling constraints.
2. Compile the dropout-free surface code using the modified objective.
3. Check whether the solver returns one of the four symmetry-related canonical syndrome-extraction circuits identified by Anker–Debroy [V, §V].
4. Record the objective value, circuit depth, solver status, and runtime.

**Acceptance criterion.** The ported objective must reproduce one of the expected canonical surface-code circuits before being evaluated on BB codes.

### T10b. Objective-term mapping

The Anker–Debroy objective is a weighted combination of terms that reward useful measurements and penalize undesirable scheduling features.

For each term, identify its equivalent in ACID's internal representation of quasi-stabilizers, rooted spanning trees, and contraction timings.

| Term | Symbol | Weight |
|---|---|---|
| Deterministic measurements | $m$ | $-1$ |
| Skip twice | $s_2$ | $6$ |
| Skip thrice | $s_3$ | $5$ |
| Alignment | $a$ | $12$ |
| Basis changes | $b$ | $2$ |

Weights are taken from Anker–Debroy Eq. 4 [V].

**Procedure.**

1. Define each objective term mathematically using quantities available in ACID's representation.
2. Determine whether the term can be evaluated without introducing assumptions specific to LUCI's surface-code geometry.
3. Pay particular attention to the *stretch* contribution within the alignment term, whose definition may depend on geometric features absent from ACID's more general framework.
4. Record which terms can be implemented exactly, which require modification, and which cannot be reliably transferred.

**Output.** An explicit term-by-term mapping between the Anker–Debroy objective and ACID's internal representation, including any limitations or unsupported terms.

### T10c. Objective–LER correlation (BB-144)

1. Use BB-144 with hexagonal connectivity and select single-coupler dropout configurations from T3.
2. Generate multiple candidate schedules using independent, time-limited ACID solves.
3. For each circuit, calculate its value under the ported objective and estimate its LER using the shared reproduction noise model.
4. Compute the Spearman rank correlation coefficient between objective value and LER:

   $$
   \rho_s=\operatorname{corr}\!\left(
   \operatorname{rank}(O),
   \operatorname{rank}(\mathrm{LER})
   \right).
   $$

   Here, $O$ denotes the ported objective value. Report the correlation and its uncertainty, and compare circuits within each dropout configuration to avoid conflating schedule quality with dropout severity.

**Output.** Objective values, LER distributions, circuit depths, solver runtimes, and Spearman correlations across candidate schedules.

**Decision rules.**

- Port only objective terms that can be consistently defined and validated within ACID.
- If the modified objective reproduces the canonical surface-code circuits and lower objective values are reliably associated with lower BB-code LER, consider adopting it as an improved scheduling heuristic.
- If the objective poorly predicts LER, retain ACID's original objective and document which terms fail to generalize.
- Only prioritize full integration into the MVP if T3/T3b demonstrate that the expected benefit justifies the additional implementation and computational cost.

**Important distinction.** A lower optimization-objective value does not automatically imply a lower logical error rate. T10c must establish whether the objective is a useful predictor of actual circuit performance.
<!--
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
