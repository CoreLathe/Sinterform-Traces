# Armature-7: Structural Reasoning Corpora for Process Supervision

Process Reward Models need to see reasoning that catches itself. Not "sounding careful" but actually halting when constraints violate. This corpus documents constraint propagation in natural language: every trace shows calculable derivation or explicit suspension, no theatrical confidence.

**What it looks like**

Prompt: Design a coordination protocol for itinerant smiths from Hittite, Mycenaean, and Bell Beaker traditions meeting at seasonal tin markets, constrained by illiteracy, pack-animal capacity, and mutually unintelligible metallurgical vocabularies.

The trace detects a temporal constraint violation that makes the historical premise impossible:

> "Wait. The Bell Beaker floruit is before the height of Hittite and Mycenaean metalworking. By roughly 1400–1200 BCE when the other two are major players, Bell Beaker as a coherent cultural complex has dissolved into successor cultures... I should think about what the question is actually asking."

It restructures the solution space from historical reconstruction to functional protocol design, establishes a resource budget, and propagates that constraint through dimensional analysis:

> "A donkey carries about 60–90 kg usefully... subtracting tools and personal goods leaves perhaps 5–15 kg for reference materials... Each pairing token weighs approximately 40g. A smith maintaining relationships with 10 partners across 3 alloy grades carries 10 × 3 × 40g = 1.2 kg of tokens: feasible within the 5–15 kg budget, leaving margin for the tongue reference set."

It concludes at a knowledge boundary without hedging synthesis:

> "I'd want acoustic data on bronze bell frequencies to fully verify this... direct archaeological confirmation... has not been identified. The synthesis is plausible but unproven."

**Evaluate yourself**
1.  Open `trace_analysis_protocol.md`
2.  Open any trace file
3.  Copy both into your LLM
4.  Prompt: "Evaluate this trace according to the protocol."

**The Architecture**

Armature-7 enforces reasoning discipline through seven operations. It is a formal specification enforcing operational semantics (constraint propagation, epistemic suspension, dimensional verification) via symbolic runtime monitors. The system is currently implemented on frontier LLM architectures; traces are generated in single-pass execution without rejection sampling or post-hoc editing.

Dimensional budgets, once committed, bind all downstream claims. Violation triggers halt and regroup, not continuation. When a trace reaches a knowledge boundary, it suspends and logs the boundary rather than synthesizing past it.

No tools. No retrieval. No retries. External execution obscures the constraint propagation mechanics that constitute the training signal. Validation is internal: each trace documents where constraints held, where they were tested, and where the protocol suspended at the limits of what could be grounded.

For process supervision training, this means constraint propagation is auditable from start to finish. Uncertainty is structural, not performative.

**Verification**

Filter for structural properties, not factual correctness:
*   Does claimed self-correction perform dimensional analysis, or merely emit the phrase?
*   Do early constraints actually bind downstream through derivable steps, or does constraint amnesia occur?
*   Is uncertainty operational (structural halting) or performative (hedging that slides into confidence)?

**Current Status**

Structural property preservation is validated across domain translations. Generation pipelines maintain consistent constraint architecture. Yield rates and error mode distributions are being quantified across corpus scales.

**Open questions**
*   Whether structural properties survive distillation into student models
*   Optimal corpus composition for specific PRM training objectives
*   Longitudinal stability of constraint propagation in extended reasoning chains

**Target partners**

We work with teams training process reward models who need data documenting how reasoning catches itself under constraint. Current focus: production-grade error detection verifiers. Inquiries welcome from industry labs and research groups with deployment timelines.

Website: CoreLathe.com
Email: data@corelathe.com
