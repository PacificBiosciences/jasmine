# Overview

The HiFi methylation callers implemented in `jasmine` estimate the likelihood of DNA modifications at specific motifs. Currently, `jasmine` supports detection of 5mC at CpG motifs and 6mA at all A sites, specifically for Fiber-seq assays. Although the caller performance is strong, false positives (incorrectly detecting a modification) and false negatives (missing an actual modification) occur.

When interpreting methylation calls, consider the known modifications in the sample to avoid mistaking false positives for true modifications or assuming false negatives mean the absence of a modification. For example:
* *PCR-amplified DNA*: DNA from PCR lacks methylation marks. Any detected 5mC or 6mA in an amplicon library should be treated as false positives and evaluated against the known false positive rates of the `jasmine` callers.
* *Human genomes*: Humans do not have functional 6mA methyltransferases and lack native 6mA methylation. Detected 6mA in human whole-genome samples should be treated as false positives unless otherwise justified.

# Accuracy

The accuracy of `jasmine v2.4` callers for individual sites on individual reads in a typical library is summarized below.  `jasmine v2.4` is used in Revio `13.3` and Vega `1.0` instrument software.

| Caller     | False positive rate (FPR) | False negative rate (FNR) | 
| ---------- | ------------------------- | ------------------------- | 
| 5mC at CpG | 7%                        |  7%                       | 
| 6mA        | 0.6%                      |  10% (for Fiber-seq)      |


# Examples

The Revio and Vega systems generate a *Methylation Summary* report for each SMRT Cell. This report provides the fraction of sites where each methylation caller predicts the presence of a modification (i.e., sites with an output probability >50%). Below are example reports and notes on how to interpret the results from three samples sequenced on the Revio system:

| Sample                 | System     | Expected marks  | 5mC at CpG | 6mA  | Notes                                                         |
| ---------------------- | ---------- | --------------- | ---------- | ---  | ------------------------------------------------------------- |
| Human WGS              | Revio 13.3 | 5mC at CpG      | 59.7%      | 0.6% | 6mA call rate is at FPR; not evidence of 6mA in sample        |
| Human Fiber-seq        | Revio 13.3 | 5mC at CpG, 6mA | 55.0%      | 9.7% | 6mA is well above FPR; good evidence of 6mA in sample         |
| Kinnex full-length RNA | Revio 13.3 | -               | 6.7%       | 0.6% | 5mC and 6mA are at FPR; not evidence of methylation in sample |
