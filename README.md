<div align="center">
    <img src="img/jasmine-logo.png" alt="jasmine logo" width="135px" align="center"/>
    <h1>Jasmine</h1>
    <p>Call select base modifications in PacBio HiFi reads</p>
</div>

## Table of contents

- [Overview](#overview)
- [Input data](#input-data)
- [Running Jasmine](#running-jasmine)
- [Supported models](#supported-models)
  - [5mC](#5mc)
  - [6mA](#6ma)
  - [5hmC](#5hmc)
- [How calls are encoded](#how-calls-are-encoded)
  - [MM tag](#mm-tag)
  - [ML tag](#ml-tag)
  - [`?` vs `.` suffix](#--vs--suffix)
- [Interpreting calls](#interpreting-calls)
  - [Present with positions](#present-with-positions)
  - [Present but empty](#present-but-empty)
  - [Absent](#absent)
- [Interpreting ML values](#interpreting-ml-values)
- [Expected biology](#expected-biology)
- [Downstream aggregation](#downstream-aggregation)
  - [MethBat](#methbat)
  - [pb-CpG-tools](#pb-cpg-tools)
- [Model training](#model-training)
- [Availability](#availability)
- [Latest version](#latest-version)
- [Changelog](#changelog)

## Overview

HiFi sequencing observes DNA synthesis in real time as a polymerase synthesizes a strand complementary to the native DNA template. The fluorescence signal identifies the canonical base (A/C/G/T), while the timing of incorporation and polymerase movement provides kinetic information that is impacted by modifications on the template strand, most commonly DNA methylation. Jasmine analyzes these kinetic signals after basecalling to estimate the probability that individual bases are chemically modified.

A few important points:

- Base modification calling with Jasmine is performed after standard A/C/G/T basecalling
- Modified bases are represented as annotations on the read, not as additional sequence letters
- Calls are probabilistic rather than binary
- Different modification types are inferred using independent models
- Results are written using standard SAM `MM`/`ML` tags

Jasmine currently supports:

| Modification | Description | Sites | MM encoding |
|---|---|---|---|
| 5mC | 5-methylcytosine | CpG cytosines | `C+m?` |
| 6mA | N6-methyladenine | Adenines on either strand | `A+a.` / `T-a.` |
| 5hmC | 5-hydroxymethylcytosine | CpG cytosines | `C+h?` / `G-h?` |

All three callers are enabled by default. The models run independently and do not share state or probabilities. Because the 5hmC and 5mC models are independent, their estimated probabilities may sum to more than 100% and should not be combined or normalized. Because the 5hmC and 5mC models use different model structures and kinetics requirements, a read may contain 5hmC calls even when 5mC calls are absent.

---

## Input data

Jasmine operates on PacBio HiFi BAM files containing kinetics tags:

- `fi` / `fp`
- `ri` / `rp`
- or single-strand `ip` / `pw`

For additional details about kinetics tags and HiFi generation, see [ccs.how](https://ccs.how/).

Reads with fewer passes than `--min-passes` (default: 2) are written to the output BAM without modification tags.

---

## Running Jasmine

Basic usage:

```sh
jasmine movie.hifi_reads.bam movie.jasmine.bam
```

### pbmod utility

Jasmine also includes `pbmod`, a utility for inspecting and comparing modification calls.

```sh
# View per-base modification calls
pbmod inspect --color movie.jasmine.bam

# Compare calls between BAMs
pbmod compare reference.jasmine.bam query.jasmine.bam
```

---

## Supported models

### 5mC

The 5mC model detects methylation at CpG sites.

In vertebrate genomes, CpG methylation is typically symmetric: if the cytosine on one strand is methylated, the complementary cytosine on the opposite strand is usually methylated as well. Because combining strands improves accuracy and robustness compared to treating each strand independently, Jasmine models the CpG as a single biological unit rather than making separate per-strand predictions. The resulting call is represented on the forward strand only using the cytosine coordinate of the CpG.

The 5mC model:

- calls CpG methylation
- reports one call per CpG dyad on the forward strand
- integrates signal from both strands into a single prediction
- requires both forward and reverse kinetics. If either strand of kinetics is missing, no `C+m?` block is produced.
- emits all eligible CpGs by default without an additional probability threshold

Downstream analyses should treat each CpG dyad as a single site and should not duplicate reverse-strand CpG calls unless required for compatibility with another tool or file format.

Future models may support strand-specific CpG methylation inference for applications involving asymmetric methylation, but current models prioritize accuracy at the CpG dyad level.

### 6mA

The 6mA model detects adenine methylation independently on each strand in the context of the Fiber-seq assay. This representation is compatible with Fiber-seq analysis tools such as [`fibertools`](https://github.com/fiberseq/fibertools-rs).

The 6mA model:

- only outputs high-confidence calls (≥0.94). Unannotated positions should be interpreted as having a low probability of modification.
- calls adenine methylation independently per strand. Kinetics are inverted relative to the called strand because polymerase kinetics are measured while synthesizing the complementary strand, indicating modifications present on the template strand:

| Kinetics | Output |
|---|---|
| reverse kinetics (`ri/rp`) | `A+a.` |
| forward kinetics (`fi/fp`) | `T-a.` |

Single-stranded `ip/pw` behaves like forward kinetics and produces `T-a.` only.

### 5hmC

The 5hmC model detects hydroxymethylation at CpG sites using a two-stage process:

1. methylated vs unmethylated
2. 5hmC vs 5mC

Unlike the 5mC model, which operates on the CpG dyad, the current 5hmC models operate on strand-specific kinetics independently.

The 5hmC model:

- detects hydroxymethylation at CpG sites
- runs with either forward or reverse kinetics
- has more permissive kinetics requirements than the 5mC model
- produces strand-specific calls
- only outputs calls above a minimum 5hmC probability threshold (≥0.5)

Requirements:

| Encoding | Required kinetics |
|---|---|
| `C+h?` | reverse kinetics |
| `G-h?` | forward kinetics or `ip/pw` |

---

## How calls are encoded

Jasmine writes modification calls using the SAM `MM` and `ML` tags.

- `MM` describes modification type and positions
- `ML` stores per-call probabilities

Any existing `MM` and `ML` tags are removed before inference.

### MM tag

The `MM` tag encodes:

- the canonical base
- the modification type
- strand/orientation
- modified positions

Example:

```text
MM:Z:C+m?,5,10;A+a.,2,0;
```

The numeric values are delta-encoded offsets between successive canonical bases of the specified type, following the SAM tags specification.

### ML tag

The `ML` tag stores probabilities corresponding to the calls listed in `MM`.

Example:

```text
ML:B:C,200,180,220
```

Each value is an integer from 0–255.

The continuous probability range `[0,1]` is mapped to integers using:

```text
scaled = min(int(probability * 256), 255)
```

An integer value `N` corresponds to probabilities in the range:

```text
[N/256, (N+1)/256)
```

Approximate interpretation:

| ML value | Probability |
|---|---|
| 255 | ~100% |
| 128 | ~50% |
| 0 | ~0% |

`ML` values correspond to `MM` entries in left-to-right order.

### Example

```text
Read  AGTCTAGACTCCGTAATTACTCGCCTAG...
C        1    2 34       5 6 78
CpG              *         *

MM:Z:C+m?,3,1,...
ML:B:C,249,4,...
```

The first CpG has a methylation probability near 97%, while the second CpG has a probability near 1.5%.

### `?` vs `.` suffix

The suffix determines how unlisted bases should be interpreted.

| Suffix | Meaning |
|---|---|
| `?` | unlisted bases were not necessarily evaluated |
| `.` | unlisted bases are implicitly unmodified |

Jasmine uses:

- `?` for 5mC and 5hmC
- `.` for 6mA

For CpG models, some sites may be skipped because they are too close to read ends or lack usable kinetics. For 6mA, unlisted adenines within the usable model window are treated as unmodified.

---

## Interpreting calls

Each encoding can appear in three states.

### Present with positions

Example:

```text
MM:Z:C+m?,5,10;
ML:B:C,200,180
```

Meaning:

- the model ran
- usable kinetics were available
- modifications passed filtering

### Present but empty

Example:

```text
MM:Z:C+m?;
```

Meaning:

- the model ran
- usable kinetics were available
- no positions were emitted

This should be interpreted as “assessed, but no reported calls,” not as missing data. 

### Absent

Example:

```text
MM:Z:A+a.,2;
```

with no `C+m?` block.

Meaning:

- the modification type could not be assessed
- required kinetics or usable sequence were missing

Absent calls should be treated as missing data rather than evidence of no modification.

This distinction allows downstream tools to differentiate between reads that were assessed but had no reported modifications and reads that could not be evaluated at all.

Reads with no `MM`/`ML` tags at all generally indicate one of:

- no usable kinetics
- passes below `--min-passes`
- no enabled model produced any output

---

## Interpreting ML values

`ML` values represent modification probabilities.

| Probability | Interpretation |
|---|---|
| >80% | strong evidence |
| 20-80% | ambiguous |
| <20% | strong evidence against modification |

Intermediate probabilities such as 40–60% should generally be interpreted as uncertain evidence rather than definitive modified or unmodified states.

Different downstream analyses may use these values differently:

| Strategy | Use case |
|---|---|
| binary threshold (~50%) | simple workflows |
| high-confidence thresholds | precision-focused analyses |
| probabilistic aggregation | statistical modeling |

---

## Expected biology

Modification calls should always be interpreted in the context of the sample and assay design.

| Sample type | Expected signal |
|---|---|
| PCR-amplified DNA | no real methylation; observed calls reflect background |
| Human WGS | CpG 5mC expected |
| Human WGS 6mA | typically background-level only |
| Fiber-seq | strong 6mA signal expected |

Additional accuracy metrics, expected false positive rates, and example interpretation guidance are available in the [jasmine accuracy documentation](https://github.com/PacificBiosciences/jasmine/blob/main/ACCURACY.md).

As with most epigenetic assays, aggregate analysis across many reads is usually more informative than interpreting individual reads in isolation.

---

## Downstream aggregation

### MethBat

[MethBat](https://github.com/PacificBiosciences/MethBat) is the recommended tool for methylation pileup and downstream analysis. It understands the Jasmine model outputs, including CpG-based 5mC and 5hmC calls, strand-specific 5hmC calls, and strand-specific 6mA calls.

MethBat supports:

- per-site methylation pileup
- differential methylation analysis
- segmentation
- cohort analysis
- rare methylation detection
- cell-type deconvolution

### pb-CpG-tools

[pb-CpG-tools](https://github.com/PacificBiosciences/pb-CpG-tools) aggregates `C+m` calls into per-CpG methylation estimates. It processes only `C+m` calls and does not support 5hmC or 6mA.

pb-CpG-tools is deprecated in favor of MethBat for new analyses.

---

## Model training

The Jasmine models are trained using supervised learning with curated positive and negative datasets.

| Model | Positive datasets | Negative datasets |
|---|---|---|
| 5mCpG | HG002 WGA + M.SssI | HG002 WGA |
| 5hmC | Ligation-based 5hmC control | HG002 WGA |
| 6mA | HG002 Fiber-seq | HG002 WGS, HG002 Fiber-seq |

For the Fiber-seq training data, labels were generated using [`fibertools predict-m6A`](https://github.com/fiberseq/fibertools-rs).

---

## Availability

The latest version can be installed via the bioconda package `pbjasmine`.

Please refer to the [official pbbioconda page](https://github.com/PacificBiosciences/pbbioconda) for information on installation, support, license, copyright, and disclaimer.

## Latest version

Version **26.1.3**: [Full changelog here](#changelog)

## Changelog

PacBio recommends the latest version for all Sequel II, Revio, and Vega datasets.

* **26.1.3**
  * New 5-hydroxymethylcytosine (5hmC) caller at CpG sites with strand-specific calls
  * New models for SPRQ-Nx chemistry (R/P2-C3/5.0-25M)
  * 6mA calling enabled by default
  * New `pbmod` multi-tool for inspecting and comparing methylation calls

* 2.7.99
  * Fix by-strand 6mA calling
