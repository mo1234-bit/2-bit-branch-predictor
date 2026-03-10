# 2-bit-branch-predictor

This report analyses the branch_predictor_2bit SystemVerilog project — a pipelined, 512-entry Branch Target Buffer (BTB) implementing a 2-bit saturating counter predictor. To quantify real-world effectiveness, the predictor is exercised against a simulated Bubble Sort inner loop executing on an 8-element array, a workload chosen for its mix of highly-predictable loop branches and data-dependent swap conditions.

<img width="850" height="91" alt="image" src="https://github.com/user-attachments/assets/cffb040a-515d-4406-ab2b-01b6ecec863c" />


The predictor eliminates 27 out of 52 mispredictions compared to the always-not-taken baseline, saving an estimated 81 pipeline flush cycles over the full sort, representing a significant IPC improvement for loop-heavy workloads.
 
# Module Architecture

# Top-Level Interface
The module is fully synchronous, driven by clk with an active-low reset (rst_n). All updates are registered on the rising clock edge; the prediction path is purely combinational, enabling a zero-latency lookup in the fetch stage.

<img width="843" height="398" alt="image" src="https://github.com/user-attachments/assets/af62c78d-ad45-4184-bd80-3183e7f86463" />


# Branch Target Buffer (BTB)
The BTB holds 512 entries, indexed by bits [10:2] of the program counter (a 9-bit direct-mapped index). Each entry stores three fields:
•	btb_valid — single-bit valid flag; 0 at reset, set to 1 on first resolution of any branch mapping to that index
•	btb_tag [31:0] — full PC of the branch that owns this entry, used to detect aliasing collisions
•	btb_target [31:0] — last known taken-target address; corrected on every execution

On reset, all valid bits are cleared and every 2-bit counter is initialised to 2'b01 (Weakly Not Taken), giving a conservative cold-start bias that avoids speculative fetches into unmapped pages.

#  2-Bit Saturating Counter
Each BTB entry carries a 2-bit saturating counter encoding four prediction states. The use of two bits provides hysteresis: a single misprediction does not immediately flip the prediction. This is particularly beneficial for inner loops whose loop-exit branch is taken every iteration except the last.

<img width="838" height="144" alt="image" src="https://github.com/user-attachments/assets/3543926a-d163-4985-904a-d268e737a92c" />


The counter is incremented (saturating at 2'b11) when a branch is taken, and decremented (saturating at 2'b00) when not taken. The prediction is taken whenever the MSB (bit 1) is 1, corresponding to the Weakly Taken and Strongly Taken states.
# Prediction Logic
The prediction is driven by a combinational always @(*) block that reads the BTB on every cycle. The prediction is taken only when all three of the following are true: (1) fetch_valid is asserted, (2) the BTB entry at the fetch index is valid, and (3) the stored tag matches the current fetch_pc. If any condition fails — including a BTB miss due to aliasing — the predictor defaults to not-taken with PC+4 as the fallback target.
# Update & Mispredict Detection
On every cycle where exec_is_branch && !stall is true, the BTB entry is unconditionally overwritten with the resolved branch details (tag, target, valid). The 2-bit counter is then incremented or decremented based on the actual outcome.
The registered mispredict output is asserted when either the predicted direction differs from the actual direction, or the branch was taken but the stored target was stale. The correct_target output simultaneously provides the pipeline with the correct fetch address for the redirect.
 
3. Benchmark Program — Bubble Sort

3.1  Rationale
Bubble sort is an ideal predictor benchmark because it contains three architecturally distinct branch types: a highly biased outer loop branch, a moderately biased inner loop branch, and a data-dependent comparison branch with near-random outcomes on unsorted data. This variety exposes the predictor's strengths (loop branches) and limits (random branches) simultaneously.
3.2  Source Program

C reference implementation (n = 8 integers, random initial order):
<img width="835" height="379" alt="image" src="https://github.com/user-attachments/assets/2c1e72d8-63f4-4bca-bb66-52e7721d6f21" />


#  Branch Characterisation
<img width="842" height="117" alt="image" src="https://github.com/user-attachments/assets/1f1dfbc7-f857-41a5-a6ea-aef7228dd656" />

 
# Performance — Without Predictor 

The baseline models a pipeline with no branch prediction: every branch is assumed not-taken and the target defaults to PC+4. This is equivalent to the state of the module immediately after reset, before any BTB entry has been populated.
# Expected Behaviour
•	Every branch that is actually taken causes a pipeline flush because the fetch unit has already fetched the fall-through instruction.
•	Branch A executes 7 times: 6 taken, 1 not-taken → 6 mandatory flushes.
•	Branch B executes 35 times: 28 taken, 7 not-taken → 28 mandatory flushes.
•	Branch C executes 28 times: 18 taken, 10 not-taken → 18 mandatory flushes.

#  Trace Excerpt — First 16 Branch Events (BTB still cold)
The table below shows the first 16 branch resolutions. Because no BTB entries are valid yet, every taken branch is a compulsory miss. The counter updates after each resolution but the BTB is still learning the access pattern.
<img width="740" height="494" alt="image" src="https://github.com/user-attachments/assets/76c29ccc-f520-4a59-bd28-3133c71dca34" />


Observation: Of the first 16 branches, 9 are mispredictions (56.3%). All taken branches in the early iterations are cold misses; the predictor can only avoid misses on Branch C when the actual outcome happens to be not-taken, aligning with the default.
# Cold Baseline Summary
<img width="605" height="173" alt="image" src="https://github.com/user-attachments/assets/aac1db91-076b-46c2-88e3-0731df42c761" />

 
# Performance — With 2-Bit BTB Predictor (Warmed)

After the first few encounters with each branch, the BTB entries are populated and the 2-bit counters have converged to a stable state reflecting the branch's bias. The following analysis covers the warmed steady-state behaviour.
#  Steady-State Counter Values
•	Branch A (outer loop, 85.7% taken): counter converges quickly to Strongly Taken (2'b11). The single not-taken event (loop exit) only pulls it down to Weakly Taken, so the next outer loop iteration re-predicts taken immediately — a single wasted cycle on exit.
•	Branch B (inner loop, 80% taken): same convergence to Strongly Taken. The 7 loop-exit events each cause one mispredict but recover quickly due to the strong taken bias.
•	Branch C (swap cond., 64.3% taken): counter oscillates between Weakly Not-Taken and Weakly Taken due to the random data pattern. This is the dominant source of residual mispredictions — a fundamental limitation of 1D predictors for data-dependent branches.

#  Trace Excerpt — Final 16 Branch Events (predictor warmed)
The table below shows branches 55–70 (the final outer loop iterations). The loop branches now predict almost perfectly; residual misses are exclusively on Branch C (swap condition).
<img width="738" height="492" alt="image" src="https://github.com/user-attachments/assets/59f703be-29c5-4c14-9abc-a93b7a3b67a0" />


Observation: Of the final 16 branches, 8 are mispredictions — all 8 are on Branch C (0x1020). Branches A and B are only mispredicted at their respective loop-exit events (a structural unavoidability). The predictor correctly handles all regular inner-loop back-edges.
# Warmed Predictor Summary
<img width="605" height="208" alt="image" src="https://github.com/user-attachments/assets/38182206-b732-4082-a1fa-2636af449cee" />

 
# Comparative Analysis

#  Overall Performance
<img width="839" height="283" alt="image" src="https://github.com/user-attachments/assets/33c3b1ce-7e9e-48ce-b1d1-a0a3554e8224" />


The 2-bit BTB predictor reduces mispredictions from 52 to 25, a 51.9% reduction. Assuming a 3-cycle branch misprediction penalty (flush + refetch + decode), this translates to 81 fewer stall cycles out of a total execution window of approximately 370 cycles, or a ~22% reduction in branch-induced stall overhead.
# Branch-by-Branch Breakdown
<img width="836" height="120" alt="image" src="https://github.com/user-attachments/assets/b4aa3b0b-a5d5-4b7f-b3b8-a9d14be5a998" />


Loop branches (A and B): The predictor delivers the most dramatic gains on both loop branches. After the first iteration, each loop back-edge converges to Strongly Taken and remains there, correctly predicting the overwhelming majority of iterations. The only unavoidable misses are the loop-exit transitions.
Swap condition (C): The near-random taken/not-taken distribution of Branch C means the 2-bit counter oscillates perpetually. This limits the predictor's benefit on this branch to a modest 10.7 percentage point improvement. Eliminating this residual source of mispredictions would require a correlation predictor (two-level adaptive) or a perceptron predictor that can learn history-dependent patterns.
 
# Design Observations & Recommendations

#  Strengths
•	Zero-cycle prediction latency: the combinational prediction path ensures predictions are available in the same cycle as fetch, preventing any added pipeline bubbles from the predictor itself.
•	Efficient BTB indexing: using bits [10:2] provides a 9-bit index covering 512 entries while skipping the two sub-word alignment bits, which would otherwise always be zero and waste index entropy.
•	Hysteretic counter: the 2-bit design avoids immediately flipping on a single anomalous branch outcome, producing high accuracy on loop back-edges that occasionally exit.
•	Stall-safe design: both the prediction and the update logic respect the stall signal, preventing phantom updates during multi-cycle stall events that could corrupt the BTB state.
•	Conservative reset state: initialising counters to Weakly Not Taken (2'b01) rather than Strongly Not Taken avoids unnecessary oscillation on early taken branches — only one taken event is needed to flip to a taken prediction.
#  Limitations & Potential Improvements
•	Aliasing: the direct-mapped BTB means two branches at PCs whose bits [10:2] collide will evict each other. For a 512-entry BTB this is relatively rare in most workloads, but could be mitigated with set-associativity or PC XOR hashing using additional bits.
•	Branch C (data-dependent): the 2-bit predictor cannot learn the data pattern driving the swap condition. A two-level adaptive predictor (e.g., gshare) that XORs global branch history with the PC index would allow the predictor to correlate the swap outcome with preceding comparisons.
•	No return address stack (RAS): function-call/return branches (not present in this benchmark) are poorly served by a BTB alone. Adding a small hardware RAS would dramatically improve prediction accuracy for call-intensive programs.
•	BTB target only updated on taken branches: the current implementation always writes exec_actual_target even when the branch is not taken, which technically stores a meaningless target for not-taken branches. While benign (the target is only used when predict_taken=1), filtering writes to taken-only events would reduce unnecessary BTB churn in aliased-entry scenarios.
#  Simulation Validity
The simulation uses a seed-42 random generator for Branch C outcomes (p=0.55 taken) on an 8-element array. For larger arrays or adversarial input distributions, Branch C accuracy is expected to remain poor regardless of predictor sophistication, as the swap condition is fundamentally input-dependent. Loop branches A and B are deterministic by construction and their results generalise directly to any array size — accuracy will be similarly high for any n.




# Vivado synthesis
The design was synthesized on Artix 7 xc7a35tcsg324-1 
# Utilization
<img width="836" height="539" alt="image" src="https://github.com/user-attachments/assets/59d418bf-e316-4c7b-923b-b1a74f039ec4" />

 
  
9. Conclusion

The branch_predictor_2bit module is synthesisable 2-bit BTB predictor that delivers substantial misprediction reduction on loop-dominated workloads. Against the Bubble Sort benchmark, the predictor cuts the misprediction rate from 74.3% to 35.7%, saving 81 pipeline cycles over the full sort execution.
The design's primary limitation is its inability to handle data-dependent branches with near-random outcomes. Future enhancements — particularly a global history register feeding a gshare or tournament predictor — would address this weakness and push the overall accuracy well above 80% on general workloads.
For loop-heavy embedded or signal-processing workloads similar to the Bubble Sort benchmark, the current design provides an excellent cost-performance trade-off within a compact 512-entry BTB footprint.
