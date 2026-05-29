# Overview

The HiFi methylation callers implemented in `jasmine` estimate the likelihood of DNA modifications at specific motifs. `jasmine v26.1` supports detection of 5mC and 5hmC at CpG motifs and 6mA at all A sites, specifically for Fiber-seq assays. Although the caller performance is strong, false positives (incorrectly detecting a modification) and false negatives (missing an actual modification) occur.

When interpreting methylation calls, consider the known modifications in the sample to avoid mistaking false positives for true modifications or assuming false negatives mean the absence of a modification. For example:
* *PCR-amplified DNA*: DNA from PCR lacks methylation marks. Any detected methylation in an amplicon library should be treated as false positives and evaluated against the known false positive rates of the `jasmine` callers.
* *Human genomes*: Humans do not have functional 6mA methyltransferases and lack native 6mA methylation. Detected 6mA in human whole-genome samples should be treated as false positives unless otherwise justified.

# Accuracy

The accuracy of `jasmine v26.1` callers for individual sites on individual reads in a typical library is summarized below.

| Caller      | False positive rate (FPR) | False negative rate (FNR) |
| ----------- | ------------------------- | ------------------------- |
| 5mC at CpG  | 11%                       |  11%                      |
| 5hmC at CpG | 0.9%                      |  20%                      |
| 6mA         | 0.6%                      |  10% (for Fiber-seq)      |


# Examples

The Revio and Vega systems generate a *Methylation Summary* report for each SMRT Cell. This report provides the fraction of sites where each methylation caller predicts the presence of a modification (i.e., sites with an output probability >50%). Below are example reports and notes on how to interpret the results from three samples sequenced on the Revio system:

| Sample                 | System     | Expected marks  | 5mC at CpG | 5hmC at CpG | 6mA   | Notes                                                         |
| ---------------------- | ---------- | --------------- | ---------- | ----------- | ----- | ------------------------------------------------------------- |
| Human WGS              | Revio 13.5 | 5mC at CpG      | 60.0%      | 0.4%        | 0.4%  | 5hmC and 6mA call rates are at FPR; not evidence of 6mA or 5hmC in sample        |
| Human Fiber-seq        | Revio 13.5 | 5mC at CpG, 6mA | 53.5%      | 0.5%        | 14.4% | 6mA and 5mC are well above FPR; good evidence of 5mC and 6mA in sample; 5hmC is at FPR         |
| Kinnex full-length RNA | Revio 13.5 | -               | 13.5%      | 0.1%        | 0.4%  | All marks are at or near FPR; not evidence of methylation in sample |

