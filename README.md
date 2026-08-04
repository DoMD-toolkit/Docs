<div align="center">
```text
 ██████████            ██████   ██████ ██████████  
░░███░░░░███          ░░██████ ██████ ░░███░░░░███ 
 ░███   ░░███  ██████  ░███░█████░███  ░███   ░░███
 ░███    ░███ ███░░███ ░███░░███ ░███  ░███    ░███
 ░███    ░███░███ ░███ ░███ ░░░  ░███  ░███    ░███
 ░███    ███ ░███ ░███ ░███      ░███  ░███    ███ 
 ██████████  ░░██████  █████     █████ ██████████  
░░░░░░░░░░    ░░░░░░  ░░░░░     ░░░░░ ░░░░░░░░░░   
```
</div>

# DoMD-Toolkit: System Architecture and Global Workflow

This document serves as the technical manual for the DoMD-Toolkit. This comprehensive suite of computational tools is engineered to bridge the gap between abstract chemical definitions, coarse-grained (CG) reaction simulations, and fully parameterized all-atom (AA) molecular dynamics systems.

The toolkit currently comprises three specialized engines, accessible via the [DoMD-Toolkit WebUI](https://www.domd.today/tools):

1. **DOMD-AL:** An all-in-one reaction simulation engine driven by a bespoke Reaction DSL.
2. **DOMD-TOPO:** A topological "Chemical Compilation" engine for Coarse-Graining to Fine-Graining (CG-FG) reconstruction.
3. **OPLS-AUTOFF:** An automatic OPLS force-field parameterizer generating ready-to-use simulation topologies.
To effectively utilize the toolkit, a rigorous understanding of module interactions and the conceptual primitives defining the chemical system is required.

---

## 1. The Global Workflow

The architectural framework of the DoMD-Toolkit operates on a sequential pipeline: **Definition $\rightarrow$ Reaction $\rightarrow$ Fine-Graining $\rightarrow$ Parameterization**.

```mermaid
flowchart TD
    A[Reaction DSL config.json] -->|Defines System| B(DOMD-AL)
    B -->|Mean-field / Particle Simulation| C[CG Conformation & Reaction Path]
    
    C -->|Input| D(DOMD-TOPO)
    A_Simp[Simplified config.json] -.->|Optional Direct Input| D
    
    D -->|S-CGFG Reconstruction| E[All-Atom sdf/pdb Files]
    
    E -->|Input| F(OPLS-AUTOFF)
    F -->|Parameterization| G[Gromacs gro/top or itp Files]

```

### 1.1 The Step-by-Step Pipeline

* **Step 1: System Definition (Reaction DSL).** The protocol initiates with a `config.json` file formulated in the Reaction DSL. This configuration establishes the CG scheme, defining fundamental reactants, filler materials, and the governing topological reaction rules.
* **Step 2: Reaction Simulation (DOMD-AL).** Driven by the DSL, DOMD-AL executes CG-scale simulations utilizing either mean-field approximations or particle-based engines. It evaluates spatial candidates, applies stochastic reaction probabilities, and produces a dynamic chronological sequence of structural transformations. The ultimate output comprises an equilibrated CG conformation alongside a detailed reaction trajectory.
* **Step 3: Fine-Graining (DOMD-TOPO).** The resulting CG conformation is transferred to DOMD-TOPO. This engine employs a SMILES/SMARTS-driven framework (S-CGFG) to deterministically backmap the coarse-grained topology into a fully coordinate-embedded AA structure.
* **Step 4: Force-Field Assignment (OPLS-AUTOFF).** Finally, the AA structure is processed by OPLS-AUTOFF. This module assigns OPLS force-field parameters—augmented by BOSS database heuristics and machine learning inference—to output standard GROMACS `.gro` and `.top`/`.itp` files.

---

## 2. System Definitions & Architectural Primitives

To precisely model reaction kinetics and topology, the Reaction DSL relies on a strict set of conceptual primitives.

### 2.1 Core Entities

| Entity | Definition & Structural Behavior |
| --- | --- |
| **Reactant** | The fundamental chemical unit. It is defined via a standard `smiles` string (for complete molecules) or a `smarts` string (for molecular fragments). Each reactant must specify a `max_valence` parameter to establish the theoretical upper bound of permissible reaction edges. |
| **Filler** | A specialized composite construct utilized to model complex topological entities (e.g., nanoparticles, crosslinkers). Architecturally, a filler is resolved into an **unreactive central core** conjugated to multiple **reactive arms**. These arms reference a specific `type` of a standard reactant, thereby inheriting its structural attributes and integrating into its corresponding dynamic node pools. |
| **Reaction** | The topological rule governing inter-entity connectivity. Reactions are formulated using RDKit-compatible SMARTS templates (e.g., `[C:1].[N:2]>>[C:1][N:2]`), which dictate the precise mechanism of edge addition to the CG graph following a successful chemical event. |

### 2.2 System State & Graph Dynamics

* **Slot Alignment:** This serves as the universal coordinate system within the DSL. Indices defined via SMARTS atom mapping (e.g., `:1`, `:2`) strictly correlate with the reactant list, candidate nodes, and designated type changes.
* **Valence Constraints:** The `max_valence` parameter operates as a strict upper limit for reaction-induced edges. Pre-existing structural bonds, such as those linking a filler core to its respective arms, do not deplete this reactive valence capacity.
* **General vs. Radical Mechanisms:**
* *General reactions* involve stochastic sampling from the aggregate pool of nodes matching the requisite type.
* *Radical reactions* mandate strict pool segregation: initiating slots are exclusively drawn from an `active` node pool, whereas target slots are selected from an `inactive` pool.
* **Type Changes:** Reactions may be deterministically coupled with `type_changes`. Upon the acceptance and logging of a primary reaction within the reaction path, any associated type transitions (e.g., Node B converting to Node C) are executed sequentially as subsequent pseudo-time events.



---

## 3. Tool Synergy & DSL Flexibility

The DoMD-Toolkit is engineered for modularity, accommodating users who require complete kinetic simulations as well as those who possess pre-equilibrated structural data.

### 3.1 DOMD-AL: The Full State Machine

For execution within DOMD-AL, a comprehensive DSL specification is mandatory. The computational engine relies on a complete state machine—encompassing dynamic node pools, strict valence tracking, intrinsic probabilities, and customizable spatial algorithms (`candidate_fn` and `prob_fn`)—to accurately integrate the simulated reaction kinetics over discrete time steps.

### 3.2 DOMD-TOPO: The Simplified Configuration

For systems with a predefined CG scheme and a corresponding CG conformation (e.g., derived from external molecular dynamics engines), formulating a complex DSL outlining reaction kinetics is unnecessary.

In the context of **DOMD-TOPO**, the `config.json` is significantly condensed, requiring only:

* A `reactants_config` dictionary mapping active CG bead types to their constituent AA SMILES representations.


* A simplified `reaction_template` detailing the topological SMARTS templates that govern spatial bead connectivity.



Functioning strictly as a "Chemical Compiler," DOMD-TOPO bypasses dynamic simulation semantics (such as reaction probabilities and active/inactive state tracking). In instances where empirical chronological reaction order pathways are omitted, DOMD-TOPO defaults to a Breadth-First Search (BFS) algorithm to deduce fundamental connection pathways and reconstruct the fully atomistic topological graph.

---

## 4. DOMD-AL Example Configuration

The following `config.json` (in `Examples/domd_al_test.zip`) demonstrates a hybrid system featuring radical polymerization and a nanoparticle filler acting as a crosslinking hub.

### 4.1 Reactants

The `reactants` array initializes the fundamental chemical species.

| Name | Formula | Properties | System Function |
| --- | --- | --- | --- |
| **I** | `CC` | `N=5`, `max_valence=1`, `activate=5` | **Initiator:** Five molecules are instantiated and assigned to the `active` pool to propagate radical reactions. Valence is restricted to 1. |
| **P** | `CC` | `N=10`, `max_valence=2` | **Monomer:** Ten molecules originate in the `inactive` pool. A `max_valence` of 2 permits linear chain propagation. |
| **R** | `[N:1]([H:2])[H:3]` | `N=2`, `max_valence=1`, `activate=0` | **Reactive Linker:** Molecules act as reactive sites. They integrate with filler structures to dictate specific topological limits. |

### 4.2 Fillers

The `fillers` section dictates spatial constraints and complex geometric boundaries.

```json
"fillers": [
  {
    "name": "SiO2",
    "N": 1,
    "file": "core.pdb",
    "mappings": [...]
  }
]

```

* **Structural Parsing:** The filler is partitioned into a non-reactive central core and four reactive exterior arms mapped to specific coordinate indices.
* **Type Inheritance:** Each arm is assigned `"type": "R"`, inheriting its structural properties and integrating directly into the global dynamic node pool for reactant `R`.



```mermaid
graph TD
    C["SiO2 Filler Center (Non-reactive)"] --- A1["Arm cg_id:1 (Type: R)"]
    C --- A2["Arm cg_id:2 (Type: R)"]
    C --- A3["Arm cg_id:3 (Type: R)"]
    C --- A4["Arm cg_id:4 (Type: R)"]
    
    style C fill:#f9f,stroke:#333,stroke-width:2px
    style A1 fill:#bbf,stroke:#333
    style A2 fill:#bbf,stroke:#333
    style A3 fill:#bbf,stroke:#333
    style A4 fill:#bbf,stroke:#333

```

### 4.3 Reaction Rules

The `reactions` array establishes topological rules and state transitions.

#### A. Radical Mechanisms (Initiation and Propagation)

```json
{
  "name": "IP-radical",
  "kind": "radical",
  "reactants": ["I", "P"],
  "activation": { "from": 0, "to": 1 }
}

```

* **Mechanism Execution:** The `"radical"` designation enforces strict pool partitioning. Slot 0 (`I`) is sampled from the active pool; Slot 1 (`P`) is sampled from the inactive pool.
* **Active State Transfer:** Following a successful reaction, the `"activation"` block shifts the radical state from Slot 0 to Slot 1.


```mermaid
sequenceDiagram
    participant Active Pool
    participant Inactive Pool
    participant System State
    
    Active Pool->>System State: Slot 0 (Reactant I)
    Inactive Pool->>System State: Slot 1 (Reactant P)
    System State-->>System State: Form I-P Topological Edge
    System State->>Inactive Pool: Slot 0 (I) loses active state
    System State->>Active Pool: Slot 1 (P) gains active state

```

#### B. General Mechanisms and Explicit State Logging

```json
{
  "name": "PP-general",
  "reactants": ["P", "P"],
  "type_changes": [ { "node": 0, "to": "P" } ]
}

```

* **Isomorphic Transition:** Instructing Node 0 (type `P`) to transition to type `"P"` forces the deterministic generation of a `TypeChangeEvent(from=P, to=P)`. This pattern explicitly records a coarse-grained coupling event in the Reaction Path without generating superficial chemical types.



```json
{
  "name": "RP-general",
  "reactants": ["R", "P"],
  "smarts": "[N:1][H:3].[CH3:2]>>[N:1][C:2].[H:3]",
  "prod_idx": [0],
  "intrinsic_probability": 0.1,
  "type_changes": [ { "node": 1, "to": "P" } ]
}

```

* **Filler Crosslinking and Product Indexing:** This rule dictates topological linkage between the `R` filler arms and the `P` matrix. The `prod_idx: [0]` parameter instructs the compiler to strictly evaluate the primary RDKit product molecule to register the resulting coarse-grained edges.

## 5. DOMD-TOPO Example: Static Topological Reconstruction

To delineate the operational modality of DOMD-TOPO when decoupled from dynamic simulations, the following `config.json` (in `Examples/au_peg_au.zip`) illustrates a pure topological reconstruction. This configuration is deployed when a user provides a pre-equilibrated coarse-grained configuration (e.g., derived from external particle engines) but lacks the explicit chronological reaction history. Under these conditions, DOMD-TOPO functions strictly as a "Chemical Compilation" engine, utilizing a Breadth-First Search (BFS) algorithm to infer connection pathways and reconstruct the all-atom (AA) graph.

### 5.1 System Initialization and Reactants

In this static compilation mode, the DSL discards dynamic state parameters (such as `activate`, `max_valence`, or discrete node generation limits `N`). The objective is solely to establish the chemical identity of the CG beads mapping to the provided coordinate file.

```json
"cg_topology_file": "out_au_peo_au_large.xml",
"reactants": [
    {
        "name": "PEO",
        "smiles": "OCCO"
    },
    {
        "name": "Au_G",
        "smarts": "[Au]"
    }
]

```

* **Topological Input:** The `cg_topology_file` explicitly references an external data structure (`out_au_peo_au_large.xml`) containing the requisite 3D spatial coordinate vectors and structural rigid-body grouping indices for the CG system.
* **Chemical Descriptors:** The `reactants` array defines the baseline molecular identities. The polyethylene oxide (`PEO`) matrix is defined via its standard SMILES string, while the reactive gold surface grafting sites (`Au_G`) are defined via an elemental SMARTS descriptor.

### 5.2 Filler Constraints and Structural Mapping

The `fillers` section introduces the rigid geometry of gold nanoparticles (`Au`) into the topological reconstruction.

```json
"fillers": [
    {
        "name": "Au",
        "file": "au.pdb",
        "mappings": [
            { "cg_id": 0, "atom_idx": [0], "type": "Au_G" },
            { "cg_id": 18, "atom_idx": [18], "type": "Au_G" }
            // ... additional mappings ...
        ],
        "filler_idx": [0]
    }
]

```

* **Explicit Anchoring:** Unlike the dynamic DOMD-AL workflow where arms react randomly, this configuration strictly maps predefined CG spatial nodes (`cg_id`) to exact atom indices (`atom_idx`) on the input `au.pdb` structure.
* **Type Designation:** Selected atoms on the gold nanoparticle surface are explicitly designated with `"type": "Au_G"`. This instructs the S-CGFG engine to treat these specific coordinates as valid structural anchoring points for the polymer matrix during the subsequent embedding phase.

### 5.3 Static Reaction Templates and BFS Fallback

The `reactions` block in this configuration is stripped of kinetic semantics (e.g., `intrinsic_probability`, `kind`, `activation`). It functions solely as a structural template library to direct localized topological assembly.

```json
"reactions": [
    {
        "name": "a",
        "reactants": [["PEO", "PEO"]],
        "smarts": "[C:1][O:2].[O:3][C:4]>>[C:1][O:2][C:4].[O:3]",
        "prod_idx": [0]
    },
    {
        "name": "b",
        "reactants": [["Au_G", "PEO"]],
        "smarts": "[Au:1].[O:2][C:3]>>[Au:1][O:2][C:3]",
        "prod_idx": [0]
    }
]
```

* **Template Mechanisms:**
* Template `a` dictates the dehydration condensation linkage between two `PEO` monomers, relying on explicit map identifiers to define the reactive centers.
* Template `b` delineates the grafting coordination between a gold surface site (`Au_G`) and a `PEO` oxygen atom.
* **The Breadth-First Search (BFS) Solver:** Because the input XML file provides the final graph connectivity but lacks the chronological historical order of bond formation, the topological compiler must determine a mathematically valid sequence to iteratively construct the AA configuration.
* **Algorithmic Resolution:** The engine implements an implicit BFS sorting algorithm to systematically infer default connection pathways across the macroscopic crosslinked network. The BFS systematically queries the static templates (`a` and `b`) to stitch the corresponding monomer subgraphs. Concurrently, it maintains a unified Reacted Atom Set to enforce state verification, ensuring that valency boundaries are not violated during the deterministic backmapping of abstract graph connections into physical Cartesian coordinates.
* **SMARTS Execution & Product Indexing:** The SMARTS string explicitly delineates a hydrogen atom (`[H:3]`) dissociating from the nitrogen to facilitate the target `N-C` bond formation. The `prod_idx: [0]` parameter instructs the compilation engine to strictly evaluate the primary product molecule generated by the RDKit backend when registering the resultant coarse-grained edges.

![au_peg_au](./images/au_peg_au.png)
