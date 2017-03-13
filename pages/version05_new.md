---
layout: page
title: Changes in R/qtl2 version 0.5
description: Changes to the data structures in R/qtl2 version 0.5
---

The internal data structures for [R/qtl2](http://kbroman.org/qtl2)
version 0.5 have undergone major changes. In addition, in several
places, QTL analysis now requires additional steps and the need to
keep track of additional objects.

The main differences are that the genotype probabilities from
`qtl2geno::calc_genoprob` and the genome scan results from
`qtl2scan::scan1` (as well as estimated QTL effects from
`qtl2scan::scan1coef` and `qtl2scan::scan1blup`) no longer include the
genetic map of marker/pseudomarker locations.

To convert data structures from the old format to the new format, we
provide functions `convert_probs()` and `convert_scan1()` in the file
[`convert_04_to_05.R`](../assets/convert_04_to_05.R).


## Genotype probabilities

First, the genotype probabilities produced by
`qtl2geno::calc_genoprob` no longer include the genetic map of
marker/pseudomarker positions, and calculating these probabilities
generally requires one extra step.

Let's load some sample data.

```{r load}
library(qtl2geno)
iron <- read_cross2(system.file("extdata", "iron.zip", package="qtl2geno"))
```

The `calc_genoprob` function no longer takes arguments `step`,
`stepwidth`, and `off_end`. Instead, you need to first call
`insert_pseudomarkers()` to insert pseudomarker positions.

```{r insert_pseudomarkers}
map <- insert_pseudomarkers(iron$gmap, step=1)
```

You then call `calc_genoprob` using that map, with the pseudomarker
positions inserted. Alternatively, you could use `iron$gmap` directly,
and do the calculations solely at the markers.

```{r calc_genoprob}
pr <- calc_genoprob(iron, map, error_prob=0.002, map_function="c-f")
```

## Genome scan

A genome scan with `qtl2scan::scan1` is performed as before, but the
result no longer includes the genetic map of marker/pseudomarker
positions, and so later use of these results generally require that
you provide **both** the results and the map.

We first get the special covariates for the X chromosome.

```{r get_x_covar}
Xcovar <- get_x_covar(iron)
```

Then we perform the genome scan.

```{r scan1}
library(qtl2scan)
out <- scan1(pr, iron$pheno, Xcovar=Xcovar)
```

We can use `qtl2scan::find_peaks` to get a summary of inferred QTL
meeting some LOD threshold, but this function needs both the `scan1`
output and the map of marker/pseudomarker positions.

```{r peaks}
find_peaks(out, map, threshold=4)
```

Similarly, to plot the results, we need to provide both the `scan1`
output and the map.

```{r plot_lod}
library(qtl2plot)
plot(out, map, ylim=c(0, maxlod(out)*1.05))
plot(out, map, lodcolumn=2, col="violetred", add=TRUE)
```

Moreover, to subset `scan1` results, you need to provide both the
`scan1` object and the map.

```{r subset_scan1}
out_selectedchr <- subset(out, map, chr=c(2, 7, 8, 9, 16))
```

Previously, when subsetting `scan1` objects with brackets (`[ ]`), it
was done with `scan1_output[chr, lodcolumns]`, but we have dropped this
feature for now. To subset by chromosome, you need to explicitly use
`subset()`, while the bracket notation just subsets the object as a
matrix `scan1_output[positions, lodcolumns]`.

## SNP association analyses

In analysis of SNP association, with a table of SNP information and
the function `qtl2scan::genoprob_to_snpprob()`, there's now an
additional step, and the SNP information table is no longer included
in the results, but needs to be provided separately.

Let's first load the `DOex` example, which is a subset of the data
from subset of the data from
[Recla et al. (2014)](http://www.ncbi.nlm.nih.gov/pubmed/24700285).

```{r load_doex}
file <- paste0("https://raw.githubusercontent.com/rqtl/",
               "qtl2data/master/DOex/DOex.zip")
DOex <- read_cross2(file)
```

Let's first do a single-QTL genome scan with the haplotype dosages.

```{r scan_doex}
pr <- calc_genoprob(DOex, error_prob=0.002)
apr <- genoprob_to_alleleprob(pr)
k <- calc_kinship(apr, "loco")
sex <- (DOex$covar$Sex == "male")*1
names(sex) <- rownames(DOex$covar)
out <- scan1(apr, DOex$pheno, k, sex)
```

Consider the peak marker on chromosome 2. We find it's physical
location:

```{r DOex_find_peak_in_Mbp}
marker <- rownames(max(out, DOex$gmap, chr="2"))
peak_Mbp <- DOex$pmap[["2"]][marker]
```

We now grab SNP information for the region.

```{r grab_snpinfo}
tmpfile <- tempfile()
file <- "https://raw.githubusercontent.com/rqtl/qtl2data/master/DOex/c2_snpinfo.rds"
download.file(file, tmpfile, quiet=TRUE)
snpinfo <- readRDS(tmpfile)
unlink(tmpfile)
```

We convert the founder genotypes to a "strain distribution
pattern" (SDP): an integer whose binary encoding corresponds to the 8
founders' genotypes.

```{r calc_sdp}
snpinfo$sdp <- calc_sdp(snpinfo[,-(1:4)])
```

Previously, `qtl2scan::genoprob_to_snpprob` would first partition the
SNPs into groups of equivalent SNPs (within the same marker interval
and having the same strain distribution pattern) and then collapse the
genotype or haplotype probabilities to SNP genotype or allele
probabilities.

The partitioning/indexing of SNPs is now a separate step:

```{r index_snps}
snpinfo <- index_snps(DOex$pmap, snpinfo)
```

This adds a few columns, indicating which SNPs are the non-equivalent
ones, and which marker intervals they belong to.

We then can call `qtl2scan::genoprob_to_snpprob()`, giving both the
probabilities and the augmented SNP info table.

```{r calc_snpprobs}
snp_apr <- genoprob_to_snpprob(apr, snpinfo)
```

We can then perform a genome scan.

```{r perform_snp_scan}
out_snps <- scan1(snp_apr, DOex$pheno, k[["2"]], sex)
```

The function `qtl2plot::plot_snpasso()` can be used to plot the
results. You need to provide both the `scan1` output and the
SNP information table (augmented by `index_snps`, with the indexing
information).

```{r plot_snp_asso}
plot_snpasso(out_snps, snpinfo)
```

To get a table of the SNPs with the largest LOD scores, again use the
function `top_snps()`, but now you need to provide the SNP information
table.

```{r top_snps}
top_snps(out_snps, snpinfo)
```