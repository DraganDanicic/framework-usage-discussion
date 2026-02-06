# Framework Choice Under a Web Component–First Architecture

## Introduction

Our microfrontend platform uses **Web Components with Shadow DOM** as the integration contract.  
This document evaluates how different frontend frameworks fit this architecture, given the **actual size, shape, and complexity of our components**.

The goal is not to compare frameworks in general, but to assess **architectural fit and long-term system cost** under our current constraints.

---

## Architectural Baseline (Agreed Constraints)

- **Web Components + Shadow DOM** are the fixed integration boundary
- Host applications and external systems consume components framework-agnostically
- Multiple squads contribute to the ecosystem
- Components are typically **small to medium**, reusable, and UI-centric
- Encapsulation and isolation are non-negotiable

---

## Key Observations

### 1. The Web Component Boundary Is the Architectural Constant

Both Angular Elements and React wrappers:
- create **real, native Web Components**
- use standard browser APIs (`customElements`, `ShadowRoot`, lifecycle callbacks)

The difference between them is **not standards compliance**, but **internal runtime coupling and framework surface area**.

---

### 2. Existing Component Complexity Already Exists

Current vanilla Web Components already implement:
- manual state management
- validation logic
- attribute ↔ DOM synchronization
- lifecycle coordination
- explicit event wiring

This means the system **already pays the complexity cost**.

The architectural question is therefore:
> *How is this unavoidable complexity managed most efficiently over time?*

---

### 3. Angular Elements Optimizes for Application-Scale MFEs

Angular Elements embeds a full Angular runtime inside each Web Component and provides:
- strong structure
- integrated tooling
- built-in DI, routing, and lifecycle management

This approach is particularly effective when:
- MFEs behave like mini-applications
- framework-level patterns are heavily used
- long-lived, structured frontends are the norm

For smaller, feature-oriented components, this level of ceremony and coupling may be disproportionate.

---

### 4. React Aligns Well With Typical Component Scope

React’s declarative, component-based model maps naturally to:
- form elements
- input validation
- conditional rendering
- reusable UI building blocks

When used **behind a strictly standardized, internally owned Web Component wrapper**, React can:
- meet the same isolation guarantees (Shadow DOM, event boundaries)
- avoid reliance on community wrapper libraries
- keep framework concerns fully internal

This allows complexity to be absorbed into the framework **without embedding application-scale structure by default**.

---

### 5. Internal Coupling Drives Long-Term Cost

Angular Elements’ deep framework integration often implies:
- tighter version coordination across MFEs
- synchronized framework upgrades
- larger blast radius for changes

A React-based approach, isolated behind Web Components, enables:
- more independent evolution of MFEs
- smaller impact of framework upgrades
- clearer separation between platform contract and internal implementation

This is particularly relevant in a multi-squad ecosystem.

---

### 6. Risk Management Is a Governance Question

Common concerns around React in enterprise environments—such as inconsistency, wrapper instability, or ecosystem sprawl—are **governance issues**, not technical limitations.

These risks can be mitigated by:
- owning a single internal Web Component wrapper
- standardizing build tooling and linting
- defining clear input/output and lifecycle conventions
- limiting allowed dependencies

With these guardrails, React does not weaken encapsulation or platform stability compared to Angular Elements.

---

## Interim Conclusion

Given that:
- the Web Component boundary is fixed,
- internal component complexity already exists,
- and most MFEs are small to medium feature slices,

the framework decision is primarily about **how internal complexity is managed over time**.

Under these conditions, a React-based implementation behind standardized Web Components can offer a better balance of **flexibility, proportionality, and long-term cost**, while Angular Elements remains a strong option for larger, application-like MFEs.

This suggests a **default-plus-exceptions** model rather than a single framework optimized for all cases.

---

## Next Steps

- Validate assumptions with a focused comparison on a representative component
- Define a standardized Web Component wrapper and tooling baseline
- Clarify criteria for when Angular Elements is the preferred option
- Decide on a default implementation approach for typical components
