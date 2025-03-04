# Copy Number Variation Analysis Pipeline

## Introduction

The copy number variation (CNV) pipeline uses either NGS or Affymetrix SNP 6.0 (SNP6) array data to identify genomic regions that are repeated and infer the copy number of these repeats. Three sets of pipelines have been used for CNV inferences. 

The first set of CNV pipelines are built onto the existing TCGA level 2 SNP6 data generated by [Birdsuite](https://www.broadinstitute.org/scientific-community/science/programs/medical-and-population-genetics/birdsuite/birdsuite) and uses the [DNAcopy](http://www.bioconductor.org/packages/release/bioc/html/DNAcopy.html) R-package to perform a circular binary segmentation (CBS) analysis [[1]](http://biostatistics.oxfordjournals.org/content/5/4/557.short). CBS translates noisy intensity measurements into chromosomal regions of equal copy number. The final output files are segmented into genomic regions with the estimated copy number for each region. The GDC further transforms these copy number values into segment mean values, which are equal to log2(copy-number/ 2). Diploid regions will have a segment mean of zero, amplified regions will have positive values, and deletions will have negative values. The resulting Copy Number Segment outputs were then used by GISTIC2 [[2]](https://genomebiology.biomedcentral.com/articles/10.1186/gb-2011-12-4-r41), [[3]](https://www.nature.com/articles/nature08822) to generate Gene-Level Copy Number Scores that powered the GDC copy number visualization before Data Release 32. GISTIC2 pipeline and output are no longer supported since Data Release 32.

The second set of CNV pipelines are built upon ASCAT [[4]](https://www.pnas.org/content/107/39/16910) algorithm for both WGS and SNP6 data. Different from the floating-point segment mean values described above, ASCAT is able to generate Allele-specific Copy Number Segment with integer copy number values, and thus the derived integer Gene-Level Copy Number. The WGS copy number analysis pipeline, called [ascatNGS](https://github.com/cancerit/ascatNgs), is described in details in [here](https://docs.gdc.cancer.gov/Data/Bioinformatics_Pipelines/DNA_Seq_Variant_Calling_Pipeline/#whole-genome-sequencing-variant-calling). The SNP6 copy number analysis pipeline, called ASCAT2, is adopted from the [example ASCAT analysis](https://github.com/VanLoo-lab/ascat/blob/master/ExampleData/ASCAT_examplePipeline.R) and generates data similar to ascatNGS.

The third CNV pipeline is only used in [AACR Project GENIE](https://www.aacr.org/professionals/research/aacr-project-genie/). The gene-level copy number scores derived by the AACR project GENIE team are remapped to new gene names in the gene model GDC uses.   

## DNACopy & GISTIC2 Pipelines
### Data Processing Steps

The GRCh38 SNP6 probe-set was produced by mapping probe sequences to the GRCh38 reference genome and can be downloaded at the [GDC Reference File Website](https://gdc.cancer.gov/about-data/data-harmonization-and-generation/gdc-reference-files).

#### Copy Number Segmentation

The [Copy Number Liftover Workflow](/Data_Dictionary/viewer/#?view=table-definition-view&id=copy_number_liftover_workflow) uses the TCGA level 2 tangent.copynumber files described above. These files were generated by first normalizing array intensity values, estimating raw copy number, and performing tangent normalization, which subtracts variation that is found in a set of normal samples. Original array intensity values (TCGA level 1) are available in the [GDC Legacy Archive](https://portal.gdc.cancer.gov/legacy-archive/) under the "Data Format: CEL" and "Platform: Affymetrix SNP 6.0" filters.

The Copy Number Liftover Workflow performs CBS analysis using the DNACopy R-package to process tangent normalized data into [Copy Number Segment](/Data_Dictionary/viewer/#?view=table-definition-view&id=copy_number_segment) files, which associate contiguous chromosome regions with log2 ratio segment means in a tab-delimited format.  The number of probes with intensity values associated with each chromosome region is also reported (probes with no intensity values are not included in this count).  During copy number segmentation probe sets from Pseudo-Autosomal Regions (PARs) were removed from males and Y chromosome segments were removed from females.

Masked copy number segments are generated using the same method except that a filtering step is performed that removes the Y chromosome and probe sets that were previously indicated to be associated with frequent germline copy-number variation.   

| I/O | Entity | Format |
|---|---|---|
| Input | [Submitted Tangent Copy Number](/Data_Dictionary/viewer/#?view=table-definition-view&id=submitted_tangent_copy_number) |  TXT |
| Output | [Copy Number Segment](/Data_Dictionary/viewer/#?view=table-definition-view&id=copy_number_segment) or Masked Copy Number Segment | TXT  |


#### Copy Number Estimation

Numeric focal-level Copy Number Variation (CNV) values were generated with "Masked Copy Number Segment" files from tumor aliquots using GISTIC2 on a project level. Only protein-coding genes were kept, and their numeric CNV values were further thresholded by a noise cutoff of 0.3:

* Genes with focal CNV values smaller than -0.3 are categorized as a "loss" (-1)
* Genes with  focal CNV values larger than 0.3 are categorized as a "gain" (+1)
* Genes with focal CNV values between and including -0.3 and 0.3 are categorized as "neutral" (0).

Values are reported in a project-level TSV file. Each row represents a gene, which is reported as an Ensembl ID and associated cytoband.  The columns represent aliquots, which are associated with CNV value categorizations (0/1/-1) for each gene.

| I/O | Entity | Format |
|---|---|---|
| Input | [Masked Copy Number Segment](/Data_Dictionary/viewer/#?view=table-definition-view&id=copy_number_segment) |  TXT |
| Output | [Copy Number Estimate](/Data_Dictionary/viewer/#?view=table-definition-view&id=copy_number_estimate) | TXT  |


#### GISTIC2 Command Line Parameters

```Shell
gistic2 \
-b <base_directory> \
-seg <segmentation_file> \
-mk <marker_file> \
-refgene <reference_gene_file> \
-ta 0.1 \
-armpeel 1 \
-brlen 0.7 \
-cap 1.5 \
-conf 0.99 \
-td 0.1 \
-genegistic 1 \
-gcm extreme \
-js 4 \
-maxseg 2000 \
-qvt 0.25 \
-rx 0 \
-savegene 1 \
(-broad 1)
```

### File Access and Availability

| Type | Description | Format |
|---|---|---|
| Copy Number Segment| A table that associates contiguous chromosomal segments with genomic coordinates, mean array intensity, and the number of probes that bind to each segment. |  TXT |
| Masked Copy Number Segment | A table with the same information as the Copy Number Segment except that segments with probes known to contain germline mutations are removed. |  TXT |
| Copy Number Estimate | A project-level file that displays gains/losses on a gene level.  Generated from the Masked Copy Number Segment files |  TXT |


## ASCAT Pipelines
### Data Processing Steps
#### Copy Number Segmentation

The [Somatic Copy Number Workflow](/Data_Dictionary/viewer/#?view=table-definition-view&id=somatic_copy_number_workflow) uses a tumor-normal pair of either SNP6 raw CEL data, or WGS data as input. The ASCAT algorithm derives allele-specific copy number segments while estimating and adjusting for tumor purity and ploidy [[4]](https://www.pnas.org/content/107/39/16910). Because there are two parental strands, the resulting Copy Number Segment or Allele-Specific Copy Number Segment files contains 3 different copy number integer values: Major_Copy_Number refers to the larger strand copy number, Minor_Copy_Number refers to the smaller strand copy number, Copy_Number is the sum of Major_Copy_Number and Minor_Copy_Number, and thus equals to the total copy number at the locus.   


| I/O | Entity | Format |
|---|---|---|
| Input | [Submitted Genotype_Array](/Data_Dictionary/viewer/#?view=table-definition-view&id=submitted_genotyping_array) |  CEL |
| Output | [Copy Number Segment](/Data_Dictionary/viewer/#?view=table-definition-view&id=copy_number_segment) or Allele-Specific Copy Number Segment | TXT  |

| I/O | Entity | Format |
|---|---|---|
| Input | [Aligned Reads](/Data_Dictionary/viewer/#?view=table-definition-view&id=aligned_reads) |  BAM |
| Output | [Copy Number Segment](/Data_Dictionary/viewer/#?view=table-definition-view&id=copy_number_segment) or Allele-Specific Copy Number Segment | TXT  |

#### Gene-Level Copy Number

Gene-level Copy Number is generated by inheriting the Copy_Number value of the residing segment in the Copy Number Segment file generated from ASCAT2 or ascatNGS workflows. 

In some occasions, one gene may overlap with more than one segments. In this case, min_copy_number is the minimum value of all segments it overlaps, max_copy_number is the maximum value of all segments it overlaps, and copy_number is calculated as the weighted (on length of overlapped regions) median of copy number values from all overlapped segments. When there is a tie (very rare), the smaller number is used. If a gene overlaps with only one segment, copy_number = min_copy_number = max_copy_number. If a gene overlaps with no segments, the gene gets empty value "" in copy_number, min_copy_number and max_copy_number.


| I/O | Entity | Format |
|---|---|---|
| Input | [Copy Number Segment](/Data_Dictionary/viewer/#?view=table-definition-view&id=copy_number_segment) or Allele-Specific Copy Number Segment | TXT  |
| Output | [Copy Number Estimate](/Data_Dictionary/viewer/#?view=table-definition-view&id=copy_number_estimate) | TXT  |


### File Access and Availability

| Type | Description | Format |
|---|---|---|
| Copy Number Segment| A table that associates contiguous chromosomal segments with genomic coordinates, and integer copy numbers. |  TXT |
| Allele-Specific Copy Number Segment| A table that associates contiguous chromosomal segments with genomic coordinates, and integer copy numbers. |  TXT |
| Copy Number Estimate | A Gene-level Copy Number file that displays integer copy number on a gene level.  Generated from Copy Number Segment or Allele-Specific Copy Number Segment files |  TXT |


## GENIE Copy Number 
### Data Processing Steps
#### Copy Number Estimation

The gene names used in the AACR GENIE Project are remapped to the gene model GDC uses. The copy number values are not changed. Please refer to AACR GENIE for the exact method how these values were derived.

| I/O | Entity | Format |
|---|---|---|
| Input | [Submitted_Genomic_Profile](/Data_Dictionary/viewer/#?view=table-definition-view&id=submitted_genomic_profile) |  TXT |
| Output | [Copy Number Estimate](/Data_Dictionary/viewer/#?view=table-definition-view&id=copy_number_estimate) | TXT  |

### File Access and Availability

| Type | Description | Format |
|---|---|---|
| Copy Number Estimate | A Gene-level Copy Number Score file that displays GISTIC2-like copy number scores on a gene level. |  TXT |



[1] Olshen, Adam B., E. S. Venkatraman, Robert Lucito, and Michael Wigler. "Circular binary segmentation for the analysis of array-based DNA copy number data." Biostatistics 5, no. 4 (2004): 557-572.

[2] Mermel, Craig H., Steven E. Schumacher, Barbara Hill, Matthew L. Meyerson, Rameen Beroukhim, and Gad Getz. "GISTIC2. 0 facilitates sensitive and confident localization of the targets of focal somatic copy-number alteration in human cancers." Genome biology 12, no. 4 (2011): R41.

[3] Beroukhim, Rameen, Craig H. Mermel, Dale Porter, Guo Wei, Soumya Raychaudhuri, Jerry Donovan, Jordi Barretina et al. "The landscape of somatic copy-number alteration across human cancers." Nature 463, no. 7283 (2010): 899.

[4] Van Loo, P., Nordgard, S. H., Lingjærde, O. C., Russnes, H. G., Rye, I. H., Sun, W. et al. "Allele-specific copy number analysis of tumors." Proceedings of the National Academy of Sciences, 107.39 (2010): 16910-16915.

