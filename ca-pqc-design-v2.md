# Cryptographic Agility & Post-Quantum Cryptography (PQC)

## 1. Executive Summary & Problem Formulation

> **Source basis:** This document preserves the original six-pillar architecture, systems, operating model, and roadmap supplied for the program. Enhancements marked through the control-plane, lifecycle, dependency-graph, policy, exception, telemetry, and maturity sections are architectural recommendations added to strengthen that source design.


The Cryptographic Agility (Crypto-Agility) Program prepares the enterprise for the transition to Post-Quantum Cryptography (PQC).

A Cryptanalytically Relevant Quantum Computer (CRQC) invalidates legacy asymmetric cryptography (RSA, ECC, Diffie-Hellman) via Shor's algorithm, while reducing the effective security strength of symmetric primitives under Grover's algorithm; the program should therefore evaluate whether the selected symmetric algorithm and parameters provide sufficient security strength for the data's required protection lifetime.

Threat actors currently conduct **Harvest Now, Decrypt Later (HNDL)** attacks against enterprise network perimeters.

### Mosca's Theorem Evaluation

```text
Data Shelf Life (X) + Migration Timeline (Y) > Quantum Threat Horizon (Z)

     10–30+ Years              3–7 Years                 ~2030–2035
          X                         Y                         Z
          └─────────────────────────┴─────────────────────────┘
                              Risk Window
                                   ↓
                    RISK: MIGRATION MAY COMPLETE AFTER THE RELEVANT QUANTUM THREAT HORIZON
```


