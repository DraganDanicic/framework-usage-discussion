## Appendix: Clarifications on Common Concerns

This appendix addresses recurring concerns raised in previous Angular vs. React evaluations, to ensure the main discussion remains focused on architectural fit rather than assumptions.

---

### 1. Web Component Wrappers and Third-Party Dependencies

React-based Web Components **do not require third-party wrapper libraries**.

- A Web Component wrapper is a small, platform-owned adapter that:
  - defines the Custom Element,
  - attaches the Shadow DOM,
  - maps attributes to props,
  - dispatches events,
  - mounts and unmounts the React tree.
- This wrapper can be implemented internally and treated as **platform infrastructure code**.
- Angular Elements also relies on a wrapper—provided by the framework—so both approaches use an adapter layer; the difference is **ownership**, not existence.

**Conclusion:**  
Wrapper stability is a governance decision, not a React limitation.

---

### 2. Runtime Configuration, Attribute Synchronization, and State Updates

Concerns around runtime attribute handling or configuration tokens are **not a capability gap** in React.

- Attribute → state synchronization is handled in the Web Component wrapper.
- React’s effect model supports reacting to runtime changes reliably.
- Angular Signals and React hooks represent **different state models**, not different levels of capability.

**Conclusion:**  
Both approaches can reliably handle dynamic configuration; the distinction is internal state semantics, not functional completeness.

---

### 3. Structure, Discipline, and Maintainability

Angular enforces structure and conventions by default via its framework and CLI.

React does not enforce structure implicitly, but this can be addressed by:
- providing a single standardized MFE template,
- defining wrapper, build, and linting conventions once,
- limiting approved dependencies.

**Conclusion:**  
Angular provides structure by framework design; React requires **explicit platform discipline**. This is a trade-off, not a deficiency.

---

### 4. Bundle Size and Tree Shaking

Reported bundle size reductions are **highly context-dependent**.

- Results depend on:
  - baseline implementation,
  - bundler configuration,
  - shared dependency strategy,
  - duplication across MFEs.
- Both Angular and React can produce optimized bundles with modern tooling.
- Meaningful comparison requires **apples-to-apples measurement** using the same bundler rules and shared dependencies.

**Conclusion:**  
Bundle size should be evaluated empirically in our context, not assumed based on framework choice alone.

---

### Closing Note

None of these points invalidate Angular Elements as a strong option for application-scale MFEs.  
They clarify that React-based Web Components, when governed and standardized, can meet the same architectural requirements while optimizing for different component scales.
