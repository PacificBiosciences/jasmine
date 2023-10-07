<img src="img/jasmine-logo.png" alt="jasmine logo" width="135px" align="right"/>
<h1 align="center">Jasmine</h1>
<p align="center">Predict 5mC in PacBio HiFi reads</p>

***

*Jasmine* predicts 5-Methylcytosine (5mC) of each CpG site in PacBio HiFi reads,
using a Convolutional Neural Network. The *jasmine* model supports the Sequel II
and Revio systems. Methylation is assumed to be symmetric between strands. The
output is reported in the forward direction with respect to the HiFi read
sequence.

## Availability
Latest version can be installed via bioconda package `pbjasmine`.

Please refer to our [official pbbioconda
page](https://github.com/PacificBiosciences/pbbioconda) for information on
Installation, Support, License, Copyright, and Disclaimer.

## Latest Version
Version **2.0.0**: [Full changelog here](#changelog)

## Input Data
Input for *jasmine* are PacBio HiFi reads with kinetics. You can generate HiFi
with kinetics on the command-line, more info on [ccs.how](https://ccs.how/):

    ccs movie.subreads.bam movie.hifi_reads.bam --hifi-kinetics

Alternatively, you can use *SMRT Link* on your HPC or define it directly in *Run
Design* for SQIIe instruments.

*Jasmine* supports [`ccs --by-strand`](https://ccs.how/faq/mode-by-strand.html)
single-strand HiFi reads with
[kinetics](https://ccs.how/faq/kinetics#what-about-single-strand-reads).

## Execution
Running *jasmine* is as simple as:

    jasmine movie.hifi_reads.bam movie.5mc.hifi_reads.bam

## Output Data
The output methylation prediction for each annotated HiFi read is encoded in the `MM` and `ML` tags,
defined in the [SAM tag specification](https://samtools.github.io/hts-specs/SAMtags.pdf).
The `MM` tag specifies the modification (5mC from *jasmine*) and to which base it applies (every CpG for *jasmine*).
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

### Example
```
Read  AGTCTAGACTCCGTAATTACTCGCCTAG...
C        1    2 34       5 6 78
CpG              *         *

MM:Z:C+m,3,1,...   # CpG sites are at C #4 (1+3) and #6 (1+3+1+1)
ML:B:C,249,4,...   # probability of methylation at the first CpG is in [249/256,250/256); second CpG is in [4/256,5/256).
```

## Run Time
*jasmine* scales nearly linear in the number of threads, achieving ~2 GBases
HiFi per minute on 16 cores. Memory footprint is very low with <100 MB per
thread.

    $ jasmine movie.hifi_reads.bam out.bam -j 16 --log-level INFO
    Reads      : 685700
    Yield      : 12.5 GBases
    Throughput : 1.8 GBases/min
    Run Time   : 6m 46s
    CPU Time   : 1h 58m
    Peak RSS   : 1.096 GB

## Training datasets
HiFi reads and subreads for true negative and true positive CpG methylation
sites are available at
https://downloads.pacbcloud.com/public/Sequel-II-CpG-training/.

The true negatives are from HG002 Whole Genome Amplification (WGA). The true
positives are from HG002 WGA + CpG Methyltransferase (M.Sssl).


## Changelog

 * **2.0.0**
   * Initial release that supports Sequel II and Revio
   * Support for [single-strand consensus reads](https://ccs.how/faq/mode-by-strand.html)
