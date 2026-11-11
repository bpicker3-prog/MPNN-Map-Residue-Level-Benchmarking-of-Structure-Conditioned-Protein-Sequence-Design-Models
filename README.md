# MPNN-Map

**Residue- and Position-Level Benchmarking for Structure-Conditioned Protein Sequence Design**

MPNN-Map is a benchmarking workflow for analyzing sequence generation from ProteinMPNN, LigandMPNN, and SolubleMPNN.

The tool accepts one or more protein backbones, performs repeated MPNN sequence generation under standardized conditions, and analyzes how the resulting sequences differ from the starting sequence at both the **amino-acid level** and the **individual residue-position level**.

Rather than evaluating sequence-design models only through overall sequence recovery, MPNN-Map identifies which amino acids and positions are retained, which are redesigned, what substitutions are produced, and how reproducible these behaviors are across independently seeded runs.

## What MPNN-Map Does

MPNN-Map provides a common workflow for:

- ProteinMPNN
- LigandMPNN
- SolubleMPNN

For each supplied backbone, the tool:

1. prepares a standardized reference structure
2. extracts the starting amino-acid sequence
3. runs the selected MPNNs
4. generates multiple sequences per model
5. compares generated sequences with the starting sequence
6. constructs complete 20 × 20 amino-acid transition matrices
7. calculates amino-acid retention and substitution behavior
8. calculates physicochemical-class transitions
9. maps mutation frequency at every sequence position
10. repeats generation across independently seeded runs
11. measures variation across replicate runs
12. performs model comparisons across matched protein datasets
13. records input and model failures without terminating the complete benchmark

## General Workflow

```text
Protein backbone(s)
        |
        v
Input preparation
        |
        +--> local PDB/mmCIF
        |
        +--> PDB identifier
        |
        v
Standardized reference backbone
        |
        +--> WT sequence
        |
        v
Prepared once and reused
across all replicate runs
        |
        +-------------------+
        |                   |
        v                   v
   Run 1 / Seed 1      Run 2 / Seed 2
        |                   |
        +---------+---------+
                  |
             [additional runs]
                  |
                  v
      ProteinMPNN / LigandMPNN /
             SolubleMPNN
                  |
                  v
          Generated sequences
                  |
        +---------+---------+
        |                   |
        v                   v
Amino-acid analysis    Position analysis
        |                   |
        v                   v
20 × 20 transitions    Per-position mutation
        |                   |
        +---------+---------+
                  |
                  v
      CSV + heatmap + statistics
```

## Input

MPNN-Map is designed to keep the user-facing workflow simple.

A researcher may provide a local protein structure:

```bash
./mpnn_map.sh --pdb my_backbone.pdb
```

Multiple proteins can be supplied using a text file:

```bash
./mpnn_map.sh --list proteins.txt
```

MPNN-Map also supports four-character PDB identifiers:

```bash
./mpnn_map.sh --pdb 1CRN
```

Local structures are used directly. PDB identifiers are retrieved when required.

The same interface can therefore be used for experimentally determined structures and researcher-provided custom protein backbones.

## Preparing Multiple Inputs

A list may contain one structure or identifier per line.

Example:

```text
/home/user/project/backbone1.pdb
/home/user/project/backbone2.pdb
1CRN
1UBQ
/home/user/project/custom_design.cif
```

MPNN-Map prepares the supplied structures before sequence generation.

Prepared references are stored in:

```text
<output>/prepared_inputs/
```

The prepared backbone and starting sequence are then reused across independent replicate runs so that only the random seed and generated sequences change between replicates.

## Supported Models

MPNN-Map currently supports:

- **ProteinMPNN**
- **LigandMPNN**
- **SolubleMPNN**

Models may be evaluated together or individually.

All three are run through a common analysis framework so that their outputs can be compared using the same residue- and position-level measurements.

## Basic Usage

Example benchmark:

```bash
./mpnn_map.sh \
    --list proteins.txt \
    --runs 3 \
    --seed 12345 \
    --num-seqs 50 \
    --temp 0.1 \
    --models "protein ligand soluble" \
    --out benchmark_results
```

Important options include:

```text
--pdb         Single protein structure or PDB identifier
--list        Text file containing multiple inputs
--runs        Number of independently seeded replicate runs
--seed        Starting random seed
--num-seqs    Number of sequences generated per model per run
--temp        MPNN sampling temperature
--models      Models to evaluate
--out         Output directory
```

Use:

```bash
./mpnn_map.sh --help
```

for the current command-line interface.

## Replicate-Aware Benchmarking

MPNN sequence generation is stochastic.

MPNN-Map therefore supports independently seeded replicate runs to determine whether observed model behavior is reproducible rather than dependent upon a single random sample.

For example:

```text
Run 1 -> seed 12345
Run 2 -> seed 12346
Run 3 -> seed 12347
```

The same prepared protein backbone and starting sequence are used for every replicate.

This creates the analysis hierarchy:

```text
protein
  |
  +-- MPNN model
        |
        +-- independent seeded run
                |
                +-- generated sequences
```

Generated sequences from one starting protein are not treated as independent protein replicates.

## Replicate Reproducibility

MPNN-Map calculates the mean behavior of each model within each independent run and then measures variation across runs.

The output:

```text
summary/across_run_summary.csv
```

contains:

- protein
- model
- metric
- number of runs
- mean across runs
- standard deviation across runs
- minimum run mean
- maximum run mean

Initial validation of the replicate workflow showed that independently seeded runs produced different generated sequences while aggregate wild-type similarity and internal model-output measurements remained generally stable across runs.

This allows users to distinguish stochastic sequence-level variation from reproducible model-level behavior.

## Amino-Acid Transition Analysis

For every generated sequence, MPNN-Map records:

```text
WT amino acid -> designed amino acid
```

For example:

```text
WT:       A G L Q M P
Design:   V G I E L P

          A>V
          G>G
          L>I
          Q>E
          M>L
          P>P
```

These transitions are aggregated into a complete 20 × 20 matrix.

Rows represent the starting amino acid.

Columns represent the generated amino acid.

The row-normalized matrix therefore represents:

\[
P(\text{designed residue} \mid \text{WT residue})
\]

Diagonal values represent exact amino-acid retention.

Off-diagonal values represent substitutions.

## Amino-Acid Outputs

For each protein and model, MPNN-Map generates:

```text
aa_transition_matrix_raw.csv
aa_transition_matrix_row_norm.csv
aa_transition_heatmap.png
```

The raw matrix contains transition counts.

The normalized matrix contains within-WT transition frequencies.

The heatmap provides a visual representation of residue-specific sequence-generation behavior.

## Position-Level Analysis

MPNN-Map also analyzes sequence generation according to **position within an individual protein**.

This answers a different question from the amino-acid transition analysis.

The amino-acid analysis asks:

> How does a model generally treat a particular amino-acid type?

The position analysis asks:

> Which positions in this specific protein does the model repeatedly retain or redesign?

For every position, MPNN-Map records:

- sequence position
- wild-type amino acid
- number of designs evaluated
- number of designs in which the position changed
- mutation rate
- counts for each of the 20 possible generated amino acids

The output is written to:

```text
position_mutation_map.csv
position_mutation_map.png
```

An example conceptual output is:

```text
Position    WT    Mutation rate
1           M        0.12
2           K        0.04
3           L        0.71
4           G        0.02
5           Q        0.83
```

Values approaching 0 indicate that the position is usually retained by the model.

Values approaching 1 indicate that the position is frequently redesigned.

These measurements describe **model sequence-generation behavior** and should not automatically be interpreted as evolutionary conservation or experimental mutational tolerance.

## Physicochemical-Class Analysis

MPNN-Map additionally groups substitutions into broader residue classes.

The current operational classification is:

| Class | Amino acids |
|---|---|
| Hydrophobic | A, V, L, I, M |
| Hydrophilic | S, T, N, Q, C, G |
| Aromatic | F, W, Y |
| Charged | K, R, H, D, E |
| Special | P |

Outputs include:

```text
class_transition_matrix_raw.csv
class_transition_matrix_row_norm.csv
class_transition_heatmap.png
```

These categories are used as a consistent benchmark classification and are not intended to represent the only possible biochemical grouping of amino acids.

## Failure Handling and Retry System

Real protein structures and computational model runs can contain edge cases.

MPNN-Map is designed so that one problematic input or model run does not terminate an entire benchmark.

Input-quality issues are reported to the user and recorded for later inspection.

When an individual MPNN run fails during the main benchmark, the successful results from the other models are preserved.

The failed protein/model combination is added to a retry queue.

After the main pass is complete, MPNN-Map retries failed model jobs once.

Conceptually:

```text
Model run
   |
   +--> success -> preserve result
   |
   +--> failure -> retry queue
                       |
                       v
                 end of main pass
                       |
                       v
                    retry
                   /     \
             success     failure
                |           |
           recovered     final failure
```

Recovered jobs and final failures are reported separately so that users can distinguish transient failures from persistent model incompatibilities.

## Input Quality Control

MPNN-Map reports structural issues when detected rather than silently modifying the input protein.

Incomplete coordinate information may therefore generate a warning while still allowing sequence-generation attempts to proceed where possible.

The goal is to preserve researcher-provided structures while transparently reporting potential input limitations.

Input-quality information is written to the benchmark output for inspection.

## PDB Retrieval

When a four-character PDB identifier is supplied instead of a local structure, MPNN-Map can retrieve the corresponding structure automatically.

Retrieved structures may be cached locally so that repeated benchmark runs do not require repeated network retrieval.

Researchers using custom PDB or mmCIF files do not require internet retrieval during the benchmark.

## Statistical Analysis

MPNN-Map performs analysis at multiple levels.

### Within Run

Generated designs are summarized within each protein/model/run combination.

Outputs include:

- mean
- median
- standard deviation
- wild-type sequence similarity
- model-reported internal confidence where available

### Across Replicate Runs

MPNN-Map measures variation in model behavior across independently seeded runs.

This provides a direct assessment of computational reproducibility.

### Across Proteins

When the same proteins are evaluated by all three models, MPNN-Map performs paired model comparisons using proteins as the independent units.

The primary comparison includes:

- Friedman test across all three models
- pairwise Wilcoxon signed-rank tests
- Holm correction for multiple pairwise comparisons

Replicate runs are averaged within each protein/model combination before the primary across-protein comparison.

Generated sequences are therefore **not treated as independent proteins**.

## Output Structure

A benchmark produces a directory similar to:

```text
benchmark_results/
|
+-- prepared_inputs/
|   +-- <protein>/
|
+-- replicates/
|   |
|   +-- run_001/
|   |   +-- runs/
|   |   +-- profiles/
|   |   +-- summary/
|   |
|   +-- run_002/
|   |
|   +-- run_003/
|
+-- summary/
    +-- run_metadata.tsv
    +-- per_run_summary.csv
    +-- across_run_summary.csv
    +-- run_level_statistical_tests.csv
    +-- primary_across_protein_tests.csv
    +-- input_qc.tsv
    +-- recovered_jobs.tsv
    +-- failed_inputs.tsv
```

Exact output contents may depend upon the selected models and number of replicate runs.

## Major Output Files

### `per_run_summary.csv`

Summarizes generated designs within each independent run.

### `across_run_summary.csv`

Measures reproducibility across independently seeded runs.

### `run_level_statistical_tests.csv`

Contains model comparisons using replicate runs as paired computational units within individual proteins where applicable.

### `primary_across_protein_tests.csv`

Contains the primary matched-protein statistical comparison across models.

### `input_qc.tsv`

Records input-quality warnings detected during preparation.

### `recovered_jobs.tsv`

Records model jobs that failed initially but completed successfully during the retry phase.

### `failed_inputs.tsv`

Records persistent input or model failures.

## What MPNN-Map Measures

MPNN-Map measures:

- wild-type sequence similarity
- model-reported internal confidence where available
- amino-acid-specific retention
- complete amino-acid transition profiles
- physicochemical-class transitions
- position-specific mutation rates
- position-specific designed-residue distributions
- variation across independent random seeds
- differences among supported MPNNs

## What MPNN-Map Does Not Measure

MPNN-Map is a **sequence-generation behavior benchmark**.

It does not independently establish:

- experimental protein folding
- thermodynamic stability
- expression
- solubility
- ligand binding
- catalytic activity
- biological function
- evolutionary conservation
- experimental mutational tolerance

Additional computational or experimental validation is required before drawing these conclusions.

## Original Benchmark Dataset

MPNN-Map was developed from a large-scale comparison of ProteinMPNN, LigandMPNN, and SolubleMPNN across a PDB-derived protein dataset.

The original analysis evaluated 953 experimentally determined protein structures.

The dataset included:

- 455 helix-rich proteins
- 193 sheet-rich proteins
- 305 mixed proteins

The original analysis used 200 generated sequences per starting structure and model at a sampling temperature of 0.1.

A total of 559,400 successful sequence generations were obtained.

The analysis showed that MPNN sequence-generation behavior varies according to model and amino-acid identity, motivating development of MPNN-Map as a reusable residue-level benchmarking framework.

## Validation

MPNN-Map has completed an end-to-end replicate validation using multiple protein backbones, all three supported MPNNs, independently seeded runs, residue-level analysis, position-level analysis, and replicate aggregation.

Initial replicate testing showed that independent seeds generated different sequence samples while aggregate model behavior remained generally stable between runs.

Larger-scale validation is ongoing.

Current software status:

**Beta**

## Sampling Temperature

The original large-scale analysis used:

```text
T = 0.1
```

Sampling temperature influences MPNN sequence generation and should therefore always be reported with benchmark results.

MPNN-Map allows temperature to be specified using:

```bash
--temp 0.1
```

Comparisons among models should use equivalent sampling conditions.

## Number of Generated Sequences

The original analysis used:

```text
200 sequences per starting structure per model
```

MPNN-Map allows researchers to choose a different number depending upon the purpose and computational scale of the benchmark.

For example:

```bash
--num-seqs 50
```

Smaller values are useful for testing and exploratory analysis.

Larger values provide more extensive sampling of model sequence-generation behavior.

## Reproducibility

Researchers reporting MPNN-Map results should record at minimum:

- input backbone dataset
- MPNN model
- model version or Git commit
- MPNN-Map version or Git commit
- sampling temperature
- number of generated sequences
- number of independent runs
- starting random seed

MPNN-Map records run metadata within the benchmark output.

## Computational Environment

MPNN-Map uses isolated Pixi environments for model inference and downstream analysis.

The workflow automatically prepares the required runtime environment and validates major dependencies before beginning the benchmark.

The software was developed and tested on Linux.

Large benchmark datasets may benefit from high-performance computing resources, while smaller datasets can be evaluated on a local workstation.

## Relationship to the Master's Thesis

MPNN-Map was developed from computational work performed for the Master's thesis:

**A Critical Evaluation of Current AI-Based Protein Design Methodologies**

The original thesis code archive contains the broader Protein-to-Sequence, Sequence-to-Protein, and combined computational pipelines.

Original thesis code:

https://github.com/bpicker3-prog/PTS-and-STP-Appendix-for-all-code

MPNN-Map is maintained separately as the reusable residue- and position-level MPNN benchmarking workflow.

## Author

**Brett Picker** 
Arizona State University  
Master of Biochemistry

**Beta**

MPNN-Map is under active validation and development.

The core workflow has successfully completed multi-protein, multi-model, independently seeded replicate benchmarking with residue-level and position-level analysis.

For reproducible published research, use a tagged release corresponding to the associated manuscript once available.
