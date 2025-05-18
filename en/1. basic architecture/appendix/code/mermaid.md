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