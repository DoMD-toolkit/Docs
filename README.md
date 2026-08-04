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

# DoMD-Toolkit: System Architecture and Global Workflow

Welcome to the DoMD-Toolkit manual. This comprehensive suite of tools is designed to bridge the gap between abstract chemical definitions, Coarse-Grained (CG) reaction simulations, and fully parameterized All-Atom (AA) molecular dynamics systems.

The toolkit currently comprises three specialized engines, accessible via the [DoMD-Toolkit WebUI](https://www.domd.today/tools):

1. **DOMD-AL:** An all-in-one reaction simulation engine driven by a bespoke Reaction DSL.
2. **DOMD-TOPO:** A topological "Chemical Compilation" engine for Coarse-Graining to Fine-Graining (CG-FG) reconstruction.
3. **OPLS-AUTOFF:** An automatic OPLS force-field parameterizer generating ready-to-use simulation topologies.

To fully leverage the toolkit, it is essential to understand how these modules interact and the primitive concepts that define a chemical system within our framework.

---

## 1. The Global Workflow

The design philosophy of the DoMD-Toolkit follows a logical pipeline: **Define $\rightarrow$ React $\rightarrow$ Fine-Grain $\rightarrow$ Parameterize**.

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

* **Step 1: System Definition (Reaction DSL).** The journey begins with a `config.json` file written in our Reaction DSL. This file declares the CG scheme, defining the foundational reactants, filler materials, and the structural reaction rules that govern them.
* **Step 2: Reaction Simulation (DOMD-AL).** Using the DSL, DOMD-AL performs simulations (via mean-field approximations or particle engines) at the CG scale. It evaluates spatial candidates, applies reaction probabilities, and generates a dynamic sequence of structural changes. The output is an equilibrated CG conformation and a detailed reaction path.
* **Step 3: Fine-Graining (DOMD-TOPO).** The CG conformation is passed to DOMD-TOPO. This engine utilizes a SMILES/SMARTS-driven framework (S-CGFG) to backmap the coarse-grained layout into a fully coordinate-embedded All-Atom (AA) structure.
* **Step 4: Force-Field Assignment (OPLS-AUTOFF).** Finally, the AA structure is fed into OPLS-AUTOFF. This tool parameterizes the molecules utilizing the OPLS force field (with support for BOSS databases and Machine Learning fallbacks) and outputs standard Gromacs `.gro` and `.top`/`.itp` files.

---

## 2. System Definitions & Architectural Primitives

To model reactions effectively, the Reaction DSL relies on a strict set of conceptual primitives. These keywords define the physics and topology of your system.

### 2.1 Core Entities

| Entity | Definition & Behavior |
| --- | --- |
| **Reactant** | The fundamental chemical unit. It is defined by either a standard `smiles` string (for complete molecules) or a `smarts` string (for molecular fragments). Each reactant specifies a `max_valence` to cap the number of reaction edges it can form. |
| **Filler** | A specialized composite structure used to model complex topological entities (like nanoparticles or crosslinkers). A filler is architecturally resolved into an **unreactive center** bound to multiple **reactive arms**. The arms reference the `type` of a standard reactant, thereby inheriting its structural properties and entering its dynamic node pools for reactions. |
| **Reaction** | The structural rule bridging entities. Reactions are defined by RDKit-compatible SMARTS templates (e.g., `[C:1].[N:2]>>[C:1][N:2]`). They dictate how edges are added to the CG graph upon a successful chemical event. |

### 2.2 System State & Graph Dynamics

* **Slot Alignment:** This is the universal coordinate system of the DSL. The indices defined in your SMARTS atom maps (e.g., `:1`, `:2`) strictly align with the reactant list, candidate nodes, and type changes.
* **Valence:** The parameter `max_valence` acts as the strict capacity limit for reaction edges. Structural bonds (like those connecting a filler center to its arm) do not consume this reaction valence.
* **General vs. Radical Mechanisms:**
* *General reactions* draw randomly from the full pool of nodes matching the required type.
* *Radical reactions* enforce strict pool separation: initiating slots must be drawn from an `active` node pool, while targets are drawn from an `inactive` pool.
* **Type Changes:** Reactions can be deterministically coupled with `type_changes`. Once a main reaction is accepted and logged in the reaction path, any associated type changes (e.g., Node B transitioning to Node C) are executed sequentially as pseudo-time events.
---

## 3. Tool Synergy & DSL Flexibility

While the Reaction DSL is robust enough to drive complex dynamic simulations in **DOMD-AL**, we designed the toolkit to be highly flexible for users who already possess structural data.

### 3.1 DOMD-AL: The Full State Machine

When running DOMD-AL, the entire DSL specification is mandatory. The engine relies on the full state machine—including node pools, valence tracking, intrinsic probabilities, and custom spatial algorithms (`candidate_fn` and `prob_fn`)—to step through the simulated reaction time correctly.

### 3.2 DOMD-TOPO: The Simplified Configuration

If you already possess a well-defined CG scheme and the corresponding CG conformation (e.g., from an external simulation), you **do not** need to write a complex DSL file mapping out reaction kinetics.

For **DOMD-TOPO**, the `config.json` is vastly simplified. You only need to provide:

* A `reactants_config` dictionary mapping the active CG bead types to their underlying All-Atom SMILES strings.
* A simplified `reaction_template` containing the topological SMARTS templates that define how these beads connect.

Because DOMD-TOPO acts purely as a "Chemical Compiler," it discards the dynamic simulation semantics (like probabilities and active/inactive states). If historical reaction order pathways are missing from your input, DOMD-TOPO will simply utilize a Breadth-First Search (BFS) algorithm to infer the default connection pathways and rebuild the All-Atom graph.
