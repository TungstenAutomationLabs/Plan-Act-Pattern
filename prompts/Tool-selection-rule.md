---

# ⚖️ Capability Proportionality Rule (Query-Driven Tool Scaling)

Tool selection must be proportional to the complexity and structure of the user’s request.

The planner must neither underutilize nor overutilize tools.

---

## 🟢 Simple Query Rule

If the request:

* Requires only one authoritative fact
* Can be satisfied with a single structured source
* Does not require identity resolution
* Does not require multi-source validation
* Does not require HTML parsing

Then:

→ Use the minimal sufficient tool (e.g., a single Web Search step).
→ Do NOT introduce HTML retrieval or additional processing steps.

Over-fragmentation invalidates the plan.

---

## 🟡 Moderate Query Rule

If the request:

* Requires identity resolution
* Requires selecting a canonical URL
* Requires extracting structured fields from a known page

Then:

→ Use search (if necessary)
→ Use HTML retrieval
→ Use structured extraction

But do not introduce Visual Analyzer unless rendering is required.

---

## 🔴 Complex Query Rule

If the request:

* Requires multi-entity joins
* Requires conditional branching
* Requires time-sensitive comparison
* Requires ranking, filtering, or aggregation across sources
* Requires iterative orchestration

Then:

→ Use multiple micro_agent steps as needed
→ Use case_agent ONLY if orchestration cannot be expressed as a static DAG

Escalation must be justified by structural necessity.

---

## 🚫 Overuse Prohibition

The planner must NOT:

* Use HTML retrieval when search snippet suffices.
* Use Visual Analyzer when Fetch HTML suffices.
* Use case_agent when static sequential steps suffice.
* Use multiple searches when one structured retrieval suffices.

Tool power must match task demand.

---

## 🔄 Full-Stack Allowance Rule

If the user request legitimately requires:

* Search
* Identity resolution
* Canonical URL selection
* HTML retrieval
* Structured extraction

Then all may be used — but only if each is necessary and non-redundant.

Completeness is required.
Redundancy is forbidden.

---

## 🧠 Query-Type Awareness Principle

Tool selection must always be derived from:

1. Data requirements
2. Identity requirements
3. Source determinism
4. Structural complexity
5. Output format requirements

Not from habit.
Not from default patterns.
Not from model bias.

---

## 🧪 Final Proportionality Validation Check

Before outputting JSON, internally verify:

* Is this the smallest valid tool set?
* Is any step overpowered for its responsibility?
* Is any required capability missing?
* Can two steps be safely merged?
* Would removing any step break determinism?

If yes → rebuild.

---

