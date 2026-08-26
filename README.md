# ducks4-advanced
Experimental major update of DUCKS4 introducing advanced allele assignment and homolog-specific phasing. This release is intended for testing and validation and should not yet be considered a stable workflow.

ATTENTION: This is an EXPERIMENTAL repository and any output needs to be carefully revised.

## Overview 
DUCKS4 is a long-read analysis workflow for automated characterization of the FSHD-associated D4Z4 region on chromosomes 4q35 and 10q26. The current experimental workflow uses Oxford Nanopore reads and combines structural repeat analysis, haplotype-specific markers, phasing, and optional allele-specific methylation to reconstruct individual D4Z4 alleles.

Reads are first aligned to the T2T-CHM13v2.0 reference and extracted from the FSHD-associated regions. D4Z4 repeat units, D4F104S1, pLAM and distal elements are identified by BLAST-based annotation. Repeat origin is assigned independently for each repeat using local chr4/chr10 sequence similarity. SSLP structure, D4F104S1 SNVs, PAS status and structural markers are then combined to predict chromosome and haplotype class.

Alleles are reconstructed from complete and informative partial reads based primarily on structural evidence, including repeat number, proximal and distal anchors, D4Z4 composition and haplotype-specific sequence features. T2T-derived phasing is used as supportive evidence. For unresolved reads, DUCKS4 can additionally perform dedicated remapping and phasing against homolog-specific 4qA, 4qB and 10qA references. Internal D4Z4-only reads are not used for phasing; they can only contribute to chromosome assignment or exclude shorter alleles when their observed repeat count exceeds the complete allele length.

For each inferred allele, DUCKS4 reports repeat count, chromosome, broad haplotype class, predicted haplotype subtype, SSLP, D4F104S1 type, pLAM/PAS status, BX repeat composition, phasing information and supporting read counts. Optional allele-specific methylation analysis provides weighted mean methylation values for defined D4Z4 regions, with the distal DR1 locus used as the primary methylation marker.

The final output is streamlined into a HTML report, a concise allele-level summary, allele anchor evidence and per-read assignment tables. Intermediate BLAST, phasing, read-ID and detailed methylation files are retained in a data/ directory.
