# Introduction
You can copy any mermaid code from this document and paste it into the [mermaid live editor](https://mermaid-js.github.io/mermaid-live-editor/) to visualize the flowchart.

# overall-architecture-flowchart
```mermaid
graph LR
    classDef process fill:#E5F6FF,stroke:#73A6FF,stroke-width:2px;

    A(arithmetic):::process --> B(arithmetic/macros.rs):::process
    A --> C(arithmetic/curves.rs):::process
    A --> D(arithmetic/fields.rs):::process
    E(plonk):::process --> F(plonk/circuit.rs):::process
    E --> G(plonk/domain.rs):::process
    E --> H(plonk/prover.rs):::process
    E --> I(plonk/srs.rs):::process
    E --> J(plonk/verifier.rs):::process
    K(polycommit):::process --> L(polycommit.rs):::process
    M(transcript):::process --> N(transcript.rs):::process
    C --> A(arithmetic):::process
    D --> A(arithmetic):::process
    F --> E(plonk):::process
    G --> E(plonk):::process
    H --> E(plonk):::process
    I --> E(plonk):::process
    J --> E(plonk):::process
    L --> K(polycommit):::process
    N --> M(transcript):::process
    A --> E(plonk):::process
    A --> K(polycommit):::process
    A --> M(transcript):::process
    E --> K(polycommit):::process
    E --> M(transcript):::process

    style A fill:#FFEBEB,stroke:#E68994,stroke-width:2px;
    style E fill:#FFEBEB,stroke:#E68994,stroke-width:2px;

    subgraph Arithmetic Module
    B
    C
    D
    end
    subgraph Plonk Module
    F
    G
    H
    I
    J
    end
    subgraph Polycommit Module
    L
    end
    subgraph Transcript Module
    N
    end
```
# macro
```mermaid
---
config:
  layout: fixed
---
flowchart TD
    A["impl_add_binop_specify_output"] -- impl --> B["Add Trait"]
    C["impl_sub_binop_specify_output"] -- impl --> D["Sub Trait"]
    E["impl_binops_additive_specify_output"] -- calls --> A & C
    F["impl_binops_multiplicative_mixed"] -- impl --> G["Mul Trait"]
    H["impl_binops_additive"] -- calls --> E
    H -- impl --> I["AddAssign Trait"] & J["SubAssign Trait"]
    K["impl_binops_multiplicative"] -- calls --> F
    K -- impl --> L["MulAssign Trait"]
    A -. Params: $lhs, $rhs, $output .-> A
    C -. Params: $lhs, $rhs, $output .-> C
    F -. Params: $lhs, $rhs, $output .-> F
    E -. Params: $lhs, $rhs, $output .-> E
    H -. Params: $lhs, $rhs .-> H
    K -. Params: $lhs, $rhs .-> K
    B -- &self + rhs, self + &rhs, self + rhs --- M1["Add Methods"]
    D -- "&self - rhs, self - &rhs, self - rhs" --- M2["Sub Methods"]
    G -- &self * rhs, self * &rhs, self * rhs --- M3["Mul Methods"]
    I -- add_assign --- M4["AddAssign Method"]
    J -- sub_assign --- M5["SubAssign Method"]
    L -- mul_assign --- M6["MulAssign Method"]
     A:::macro
     B:::trait
     C:::macro
     D:::trait
     E:::macro
     F:::macro
     G:::trait
     H:::macro
     I:::trait
     J:::trait
     K:::macro
     L:::trait
     M1:::method
     M2:::method
     M3:::method
     M4:::method
     M5:::method
     M6:::method
    classDef macro fill:#E5F6FF,stroke:#73A6FF,stroke-width:2px
    classDef trait fill:#FFF6CC,stroke:#FFBC52,stroke-width:2px
    classDef method fill:#fff,stroke:#bbb,stroke-dasharray: 5 5

```

# arithmetic-curve
```mermaid
graph LR
    classDef process fill:#E5F6FF,stroke:#73A6FF,stroke-width:2px
    
    A(Curve Trait):::process --> B(CurveAffine Trait):::process
    A --> C(Curve Implementations):::process
    B --> C
    C --> D(Projective Point Struct):::process
    C --> E(Affine Point Struct):::process
    D --> F(Arithmetic Operations):::process
    E --> F
    F --> G(Serialization and Deserialization):::process
    A --> H(Group Operations):::process
    H --> F
    
    I(Group Trait):::process --> J(group_zero):::process
    I --> K(group_add):::process
    I --> L(group_sub):::process
    I --> M(group_scale):::process
    H --> I
    
    N(get_challenge_scalar):::process --> O(best_multiexp):::process
    P(multiexp_serial):::process --> O
    O --> F
    
    Q(best_fft):::process --> R(serial_fft):::process
    Q --> S(parallel_fft):::process
    R --> F
    S --> F
    
    T(eval_polynomial):::process --> F
    U(compute_inner_product):::process --> F
    V(parallelize):::process --> F
    W(log2_floor):::process --> S
```