# MPNN-Map

**Residue- and Position-Level Benchmarking for Structure-Conditioned Protein Sequence Design**

MPNN-Map is a benchmarking workflow for analyzing sequence generation across ProteinMPNN, LigandMPNN, and SolubleMPNN. The tool accepts one or more protein backbones and measures how the models modify the starting amino-acid sequence at both the **amino-acid level** and the **individual residue-position level**.

Rather than evaluating sequence-design models only through overall sequence recovery, MPNN-Map determines:

- which amino-acid types are retained or replaced
- what substitutions each amino acid produces
- which individual positions within a protein are conserved or frequently changed
- whether substitutions retain broader physicochemical residue properties
- how sequence-generation behavior differs among ProteinMPNN, LigandMPNN, and SolubleMPNN
- how reproducible these behaviors are across independently seeded runs

## What MPNN-Map Does

For every generated sequence, MPNN-Map compares the designed amino acid with the corresponding wild-type amino acid.

Two complementary analyses are then performed.

### 1. Amino-Acid-Level Analysis

Transitions are aggregated according to amino-acid identity to produce a complete 20 × 20 transition matrix.

```text
Wild type:   A G L Q M P
Generated:   V G I E L P
              | | | | | |
Transition:  A>V G>G L>I Q>E M>L P>P
```

Rows represent the wild-type amino acid and columns represent the generated amino acid.

Diagonal values therefore represent exact residue retention, while off-diagonal values represent substitutions.

This analysis produces:

- raw amino-acid transition counts
- row-normalized 20 × 20 transition matrices
- amino-acid transition heatmaps
- residue-retention measurements
- physicochemical-class transition matrices
- physicochemical-class heatmaps

### 2. Position-Level Analysis

MPNN-Map also tracks every residue according to its position within the starting protein.

For each position, the tool records:

- residue position
- wild-type amino acid
- number of generated designs evaluated
- number of designs in which the position changed
- mutation rate
- counts for each of the 20 possible designed amino acids

For example:

```text
Position    WT    Mutation rate
1           M        0.12
2           K        0.04
3           L        0.71
4           G        0.02
5           Q        0.83
...
```

This makes it possible to distinguish two different questions:

**Amino-acid analysis:**  
How does the model generally treat glutamine, glycine, proline, or another amino-acid type?

**Position analysis:**  
Which specific positions within this particular protein does the model repeatedly preserve or redesign?

The position-level output therefore provides a direct map of where sequence redesign occurs along an individual protein backbone.

## General Workflow

```text
Protein backbone(s)
        |
        v
ProteinMPNN / LigandMPNN / SolubleMPNN
        |
        v
Generated sequences
        |
        v
WT vs. designed sequence comparison
        |
        +-------------------------------+
        |                               |
        v                               v
Amino-acid-level analysis       Position-level analysis
        |                               |
        v                               v
20 x 20 transition matrix       Per-position mutation map
        |
        v
Physicochemical-class analysis
        |
        +-------------------------------+
                        |
                        v
              CSV files + heatmaps
```

## Input

Input can be provided as:

- a local PDB structure
- a local mmCIF structure
- a four-character PDB identifier
- multiple protein structures
- a text file containing protein paths or PDB identifiers

When a PDB identifier is provided, MPNN-Map retrieves biological assembly 1 and reduces the structure to the largest standard-amino-acid protein chain before sequence generation.

Example:

```bash
./mpnn_map.sh --pdb 1CRN
```

Multiple proteins can be provided:

```bash
./mpnn_map.sh \
    --pdb 1CRN \
    --pdb 1UBQ
```

or through a list:

```bash
./mpnn_map.sh --list proteins.txt
```

## Currently Supported Models

- ProteinMPNN
- LigandMPNN
- SolubleMPNN

Models can be evaluated together or individually.

## Basic Usage

The default benchmark uses three independently seeded runs, 200 generated sequences per model per run, and a sampling temperature of 0.1.

```bash
./mpnn_map.sh \
    --list proteins.txt \
    --runs 3 \
    --seed 42 \
    --num-seqs 200 \
    --temp 0.1
```

A larger reproducibility analysis can be performed using:

```bash
./mpnn_map.sh \
    --list proteins.txt \
    --runs 5 \
    --seed 12345 \
    --num-seqs 200 \
    --temp 0.1
```

Available options include:

```text
--pdb          Protein structure or PDB ID
--list         File containing protein structures/PDB IDs
--runs         Number of independently seeded runs
--seed         Starting random seed
--num-seqs     Number of sequences generated per model per run
--temp         MPNN sampling temperature
--models       Models to evaluate
--out          Output directory
```

Run:

```bash
./mpnn_map.sh --help
```

for the complete current command-line interface.

## Replicate-Aware Benchmarking

MPNN-Map can perform multiple independently seeded sequence-generation runs.

The default configuration uses:

```text
Runs:          3
Base seed:     42
Sequences:     200 per model per run
Temperature:   0.1
```

Each run receives a different random seed.

This allows MPNN-Map to determine whether observed model behavior is reproducible across independent sequence-generation runs rather than depending upon one random sample.

Generated sequences are not treated as independent proteins during across-protein statistical testing.

When multiple proteins are supplied, replicate runs are first averaged within each protein/model combination so that the starting proteins remain the independent statistical units.

## Statistical Analysis

For datasets containing multiple proteins evaluated by all three models, MPNN-Map performs paired model comparisons.

The primary across-protein analysis includes:

- Friedman test across ProteinMPNN, LigandMPNN, and SolubleMPNN
- pairwise Wilcoxon signed-rank tests
- Holm correction for multiple pairwise comparisons

Run-level statistics are additionally generated to evaluate reproducibility across independently seeded runs.

Single-protein design-level comparisons are treated as exploratory because generated sequences from the same starting protein are not equivalent to independent protein replicates.

## Output

Benchmark results are separated into replicate runs, protein profiles, and summary statistics.

```text
mpnn_benchmark_results/
|
+-- replicates/
|   +-- run_001/
|   +-- run_002/
|   +-- run_003/
|
+-- summary/
|   +-- run_metadata.tsv
|   +-- per_run_summary.csv
|   +-- across_run_summary.csv
|   +-- run_level_statistical_tests.csv
|   +-- primary_across_protein_tests.csv
|
+-- profiles/
    +-- <protein>/
        +-- protein/
        +-- ligand/
        +-- soluble/
```

Within each protein/model profile, MPNN-Map produces outputs including:

```text
aa_transition_matrix_raw.csv
aa_transition_matrix_row_norm.csv
aa_transition_heatmap.png

class_transition_matrix_raw.csv
class_transition_matrix_row_norm.csv
class_transition_heatmap.png

position_mutation_map.csv
position_mutation_map.png
```

## Position Mutation Map

`position_mutation_map.csv` contains the complete position-level output.

Columns include:

```text
position
wt_residue
n_designs
mutation_count
mutation_rate
design_A
design_C
design_D
...
design_Y
```

The corresponding `position_mutation_map.png` visualizes the mutation rate across the complete protein sequence.

A value approaching **0** indicates that the model usually retains the wild-type amino acid at that position.

A value approaching **1** indicates that the model changes that position in most generated sequences.

This allows MPNN-Map to identify possible **sequence-constrained and sequence-variable regions** within an individual backbone.

Importantly, these values describe model sequence-generation behavior and should not automatically be interpreted as biological conservation or experimental mutational tolerance.

## Physicochemical Residue Classes

The current property-based classification is:

| Class | Amino acids |
|---|---|
| Hydrophobic | A, V, L, I, M |
| Hydrophilic | S, T, N, Q, C, G |
| Aromatic | F, W, Y |
| Charged | K, R, H, D, E |
| Special | P |

These are operational categories used for comparison and are not intended to represent the only possible biochemical classification of amino acids.

## What MPNN-Map Measures

MPNN-Map can determine:

- overall wild-type sequence identity
- model-reported internal confidence where available
- amino-acid-specific retention
- complete amino-acid substitution distributions
- physicochemical-class retention
- position-specific mutation rates
- position-specific designed-residue distributions
- variation across independently seeded runs
- differences among supported MPNNs

## What MPNN-Map Does Not Measure

MPNN-Map measures **sequence-generation behavior**.

It does not independently establish:

- experimental mutational tolerance
- evolutionary conservation
- protein stability
- protein expression
- solubility
- ligand affinity
- catalytic activity
- biological function

These conclusions require additional computational or experimental validation.

## Reproducibility

MPNN-Map uses isolated Pixi environments for MPNN inference and downstream analysis.

For reproducible reporting, researchers should record:

- input backbone dataset
- model
- model version/Git commit
- MPNN-Map version/Git commit
- sampling temperature
- number of sequences
- number of independent runs
- base random seed

Run metadata are automatically written to the benchmark output.

## Original Benchmark

MPNN-Map was developed from a large-scale comparison of ProteinMPNN, LigandMPNN, and SolubleMPNN.

The original analysis evaluated 953 experimentally determined protein structures and successfully generated 559,400 sequences across the three MPNNs.

That analysis showed that sequence-generation behavior differs according to both model and amino-acid identity, motivating development of a reusable framework capable of applying the same analysis to researcher-provided protein backbones.

## Relationship to the Master's Thesis

MPNN-Map was developed from work performed for the Master's thesis:

**A Critical Evaluation of Current AI-Based Protein Design Methodologies**

The complete original thesis computational archive is available at:

https://github.com/bpicker3-prog/PTS-and-STP-Appendix-for-all-code

The thesis repository preserves the original PTS, STP, and combined-pipeline scripts.

MPNN-Map is maintained separately as the generalized residue- and position-level benchmarking tool intended for reuse with additional protein backbones.

## Validation Status

MPNN-Map is currently being validated against the original large-scale MPNN analysis.

Validation includes reproduction of:

- wild-type sequence-identity behavior
- amino-acid transition profiles
- residue-retention patterns
- physicochemical-class profiles
- statistical model comparisons

Position-level analysis extends the original aggregate residue analysis by identifying where those sequence changes occur within individual protein backbones.

## Author

**Brett Picker**  
Arizona State University  
Master of Biochemistry


MPNN-Map is currently under active development.

For reproducible research, use the tagged release associated with the corresponding publication once available.
