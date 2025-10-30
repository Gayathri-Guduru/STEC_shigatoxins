# STECmetadetector_Shigatoxins
Bioinformatics pipeline for sequence typing and detecting Shiga toxin-producing E. coli (STEC) from metagenomic data. Identifies stx genes, O/H antigens, and virulence profiles across multiple coverage levels. It analyzes genomic coverage data for detecting Shiga toxin-producing Escherichia coli (STEC) in wastewater samples. It processes coverage files to determine whether samples are STEC or NON-STEC based on the presence of specific genes (particularly those containing "stx" in their names) using both dynamic (coverage-dependent) and static (fixed) thresholds. The pipeline operates in parallel to handle multiple samples efficiently and generates aggregated results summarizing the findings.
  
## Overview
This tool summarizes the benchmarking and evaluation of a metagenomic detection pipeline for STEC/NON-STEC isolates using a synthetic spike-in experiment. The main goal was to assess the limit of detection and pipeline performance for a wide range of isolate coverages in a realistic wastewater metagenome background. It is designed to analyze sequencing data from Shiga toxin-producing Escherichia coli (STEC) samples, specifically from wastewater. It processes coverage files to identify STEC strains by filtering genes based on coverage fractions using two modes:

- **Dynamic Mode**: Applies coverage-level-specific thresholds (e.g., 0.04 for 0.1X, 0.4 for 10X) to detect Shiga toxin (stx) genes.  
- **Static Mode**: Uses a fixed threshold (0.5) for all coverage levels.

## Dependencies
- Python 3
- fastp
- fastqc
- minimap2
- samtools
- bedtools
- coverM

## Pipeline 
The pipeline processes paired-end sequencing data to:
 - Downsample isolate and wastewater samples to various coverage levels - 0.1X, 0.5X, 1X, 2.5X, 5X, 10X, 25X.
 - Spike-in isolates into wastewater reads based on the matching coverage level samples.
 - Map reads to STEC gene database using minimap2.
     - The script uses minimap2 for aligning reads to the STEC genes database.
 - Filter alignments based on identity and coverage thresholds.
     1. Read-level filtering (CoverM)
        ```
        coverm filter --min-read-percent-identity 80 
              --min-read-aligned-percent 80 
              --bam-files {bam_file} 
              --output-bam-files {filtered_bam}
        ```
        - Filters out low-quality read alignments
        - Keeps only reads that meet BOTH criteria:
          - ≥80% identity: Read sequence matches the reference gene with 80%+ accuracy
          - ≥80% aligned: At least 80% of the read length aligns to the reference
            
      2. Gene-level filtering (Coverage Fraction) - >= 0.5
        - After calculating coverage with bedtools coverage, checks each gene
        - Keeps only genes where ≥50% of the gene length is covered by filtered reads
        - The "fraction_covered" column shows what percentage of the gene has read coverage
  Why: A gene is only considered "present" if a substantial portion is covered. This prevents false positives from reads hitting just a small fragment of a gene.
 
 - Classify samples as STEC, or NON-STEC based on gene presence.
    - EHEC: Has stx genes (stx1/stx2) AND eae gene
    - STEC: Has stx genes but no eae
    - NON-STEC: No stx genes detected

## Reference files
The reference database used is **genes.fasta**: STEC gene sequences which contains.
- **fliC** genes encode **H-antigen**. They correspond to flagellin proteins (H antigens) – 93 genes
- **wzx/wzy** genes encode for **O-antigen** and they encode proteins for O-antigen polysaccharide biosynthesis – 458 genes
- **Stx1** and **stx2** genes: Detect and subtype stx1/stx2. stx2 is more virulent and strongly associated with hemolytic uremic syndrome (HUS)- 147 genes
- **STEC** genes – Unnamed genes from STEC isolates used for typing. These might represent putative virulence genes, accessory genes, or hypothetical proteins found in a reference STEC genome - 578 genes.
- **flnA**, **flmA**, **fllA**, **flkA** genes - These genes encode alternative flagellin proteins — components of the bacterial flagellum, which forms the basis of the H (flagellar) antigen – 9 genes
- 7 house-keeping genes.
- **eae** genes.

**genes.bed**: Gene coordinates (auto-generated from FASTA)

## Input Files:

- ```coverage.csv``` contains sra_ids, coverage, read_length that lists samples to process.
- fastq files are in s3 path and the path is mentioned in the script #can be changed

## Output Files:

- Main result: cov_80_250_4.tsv
  - Final classification table with columns:
    - sample, read_length, coverage
    - stx_genes_80, stx_subtype_80

- Per-sample results (uploaded to S3):
  - {sample}_genes_coverage.tsv - Raw coverage data
  - {sample}_genes_coverage.filtered.tsv - Genes with ≥50% coverage

The script processes 10 samples in parallel, handling multiple read lengths (100/150/250/300 bp) and various coverage levels.

## Docker

```docker pull gayathriguduru/stec_pipeline:latest```

## **Conclusion**
- Dynamic thresholds greatly enhance sensitivity (TP) with only a negligible increase in false positives. The classification accuracy improves across all coverage levels, making dynamic thresholding the preferred strategy for high-confidence and comprehensive detection.








