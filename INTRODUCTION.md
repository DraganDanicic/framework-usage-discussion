## Introduction

Our microfrontend platform uses **Web Components with Shadow DOM** as the integration contract.  
This document evaluates which implementation technology best fits our **actual component scope**—small to medium, reusable UI components.

---

## What this document aims to show

- **The Web Component boundary is the architectural constant**  
  → the framework is an internal implementation detail.

- **Angular Elements and React wrappers both produce real, native Web Components**  
  → the difference lies in internal runtime coupling, not standards compliance.

- **Our existing components already contain significant manual state, validation, and DOM logic**  
  → this complexity exists regardless of framework choice.

- **The decision is therefore about how this complexity is managed long-term**  
  → not whether complexity exists at all.

- **Angular Elements is optimized for application-scale MFEs**  
  → its ceremony and runtime primarily pay off for larger, structured frontends.

- **React, behind a standardized and internally owned wrapper, can meet the same isolation guarantees**  
  → including Shadow DOM and strict input/output boundaries.

- **React can reduce boilerplate and cognitive overhead for our typical component size**  
  → without weakening encapsulation or integration safety.

- **This evaluation focuses on architectural fit and total system cost**  
  → not on developer preference or familiarity.
