<img src="img/jasmine-logo.png" alt="jasmine logo" width="135px" align="right"/>
<h1 align="center">Jasmine</h1>
<p align="center">Predict 5mC in PacBio HiFi reads</p>

***

*Jasmine* predicts 5-Methylcytosine (5mC) of each CpG in PacBio HiFi reads, using
a Convolutional Neural Network.  The *Jasmine* model supports the Sequel II and 
Revio systems.  Methylation is assumed to be symmetric between strands. The output
is reported in the forward direction with respect to the HiFi read sequence.

## Availability
Latest version can be installed via bioconda package `pbjasmine`.

Please refer to our [official pbbioconda page](https://github.com/PacificBiosciences/pbbioconda)
for information on Installation, Support, License, Copyright, and Disclaimer.

## Latest Version
Version **2.0.0**: [Full changelog here](#changelog)

## Input Data
Input for *jasmine* are PacBio HiFi reads with kinetics. You can generate HiFi
with kinetics on the command-line, more info on [ccs.how](https://ccs.how/):

    ccs movie.subreads.bam movie.hifi_reads.bam --hifi-kinetics

Alternatively, you can use *SMRT Link* on your HPC or define it directly in *Run
Design* for SQIIe instruments.

## Execution
Running *jasmine* is as simple as:

    jasmine movie.hifi_reads.bam movie.5mc.hifi_reads.bam

## Output Data
The output is adhering to the [SAM tag specification from 9. Dec 2021](https://samtools.github.io/hts-specs/SAMtags.pdf),
using `MM` and `ML` tags. It's also described in the [PacBio BAM file formats](https://pacbiofileformats.readthedocs.io/en/latest/BAM.html#use-of-read-tags-for-per-read-base-base-modifications) as

| Tag  | Type  |           Description            |
| ---- | ----- | -------------------------------- |
| `MM` | `Z`   | Base modifications / methylation |
| `ML` | `B,C` | Base modification probabilities  |

Notes for `ML`: The continuous probability range of 0.0 to 1.0 is remapped to
the discrete integers 0 to 255 inclusively. The probability range corresponding
to an integer N is `N/256` to `(N + 1)/256`.

## Run Time
*jasmine* scales nearly linear in the number of threads, achieving ~2 GBases HiFi per minute on
16 cores. Memory footprint is very low with <100 MB per thread.

    $ jasmine movie.hifi_reads.bam out.bam -j 16 --log-level INFO
    Reads      : 685700
    Yield      : 12.5 GBases
    Throughput : 1.8 GBases/min
    Run Time   : 6m 46s
    CPU Time   : 1h 58m
    Peak RSS   : 1.096 GB

## Training datasets
HiFi reads and subreads for true negative and true positive CpG methylation sites are available at https://downloads.pacbcloud.com/public/Sequel-II-CpG-training/.

The true negatives are from HG002 Whole Genome Amplification (WGA).  The true positives are from HG002 WGA + CpG Methyltransferase (M.Sssl).


## Changelog

 * **2.0.0**
   * Initial release that supports Sequel II and Revio
   * Support for single-strand consensus reads
