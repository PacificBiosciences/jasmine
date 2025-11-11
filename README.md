<div align="center">
    <img src="img/jasmine-logo.png" alt="jasmine logo" width="135px" align="center"/>
    <h1>Jasmine</h1>
    <p>Call select base modifications in PacBio HiFi reads</p>
</div>

***

`jasmine` identifies specific base modifications in PacBio HiFi reads by analyzing polymerase kinetic signatures.
The current models support detecting 5-Methylcytosine (5mC) at CpG sites and N6-Methyladenine (6mA) in Fiber-seq data.
For 5mC, the caller assumes that methylation is symmetric at the CpG site and reports methylation on the forward strand
of the read. In contrast, the 6mA caller provides strand-specific results. The 6mA calls are compatible with Fiber-seq
downstream analysis tools, such as [`fibertools`](https://github.com/fiberseq/fibertools-rs)
([Jha 2024](https://doi.org/10.1101/gr.279095.124)).

## Availability
Latest version can be installed via bioconda package `pbjasmine`.

Please refer to our [official pbbioconda
page](https://github.com/PacificBiosciences/pbbioconda) for information on
Installation, Support, License, Copyright, and Disclaimer.

## Latest Version
Version **2.4.0**: [Full changelog here](#changelog)

## Input Data
Input for *jasmine* are PacBio HiFi reads with kinetics. For more info see [ccs.how](https://ccs.how/):

## Execution
Running *jasmine* is as simple as:

    jasmine movie.hifi_reads.bam movie.methylation.hifi_reads.bam

## Output Data
The output methylation prediction for each annotated HiFi read is encoded in the `MM` and `ML` tags,
defined in the [SAM tag specification](https://samtools.github.io/hts-specs/SAMtags.pdf).
The `MM` tag specifies the modification and to which base it applies.
The `ML` tag specifies the probability of methylation at each base.

The output is also described in the [PacBio BAM file
format documentation](https://pacbiofileformats.readthedocs.io/en/latest/BAM.html#use-of-read-tags-for-per-read-base-base-modifications)
as

| Tag  | Type  |           Description            |
| ---- | ----- | -------------------------------- |
| `MM` | `Z`   | Base modifications / methylation |
| `ML` | `B,C` | Base modification probabilities  |

Notes for `ML`: The continuous probability range of 0.0 to 1.0 is remapped to
the discrete integers 0 to 255 inclusively. The probability range corresponding
to an integer N is `N/256` to `(N + 1)/256`.
The `ML` tag presents the probabilities in the order of modifications seen in the `MM` tag.

### Example
```
Read  AGTCTAGACTCCGTAATTACTCGCCTAG...
C        1    2 34       5 6 78
CpG              *         *

MM:Z:C+m,3,1,...   # CpG sites are at C #4 (1+3) and #6 (1+3+1+1)
ML:B:C,249,4,...   # probability of methylation at the first CpG is in [249/256,250/256); second CpG is in [4/256,5/256).
```

## Model training

The `jasmine` methylation models are trained using a supervised learning approach with curated positive and negative datasets.

| Model | Positive datasets  | Negative datasets         |
| ----- | ------------------ | -------------------------- |
| 5mCpG | HG002 WGA + M.SssI | HG002 WGA                  |
| 6mA   | HG002 Fiber-seq    | HG002 WGS, HG002 Fiber-seq |

For the HG002 Fiber-seq training dataset, positive and negative labels were generated using [`fibertools predict-m6A`](https://github.com/fiberseq/fibertools-rs) ([Jha 2024](https://doi.org/10.1101/gr.279095.124)).

## Changelog
 PacBio recommends the latest version (2.4.0) for all Sequel II and Revio datasets.  PacBio recommends against using older versions.

 * **2.7.99**
   * Fix by-strand 6mA calling

 * 2.4.0
   * Updated 5mCpG caller, with new model design and input features
   * New 6mA caller

 * 2.0.0
   * Initial release of jasmine 5mCpG model that supports Sequel II and Revio.
   * Support for [single-strand consensus reads](https://ccs.how/faq/mode-by-strand.html)
