
# DUCKS4-Advanced

Experimental major update of DUCKS4 introducing advanced allele assignment and based on de-novo variant fingerprinting and homolog-specific phasing. 

> 
> **ATTENTION: This is an EXPERIMENTAL repository and any output needs to be carefully revised.**
> Note: v0.1.0 may still over-split fingerprint populations in some datasets. This will be addressed in the following update. 
> 
> 

DUCKS4-Advanced extends the original DUCKS4 workflow with automated reconstruction of individual D4Z4 alleles from long-read Oxford Nanopore data.

The main aim is to resolve the four D4Z4 chromosome backgrounds at 4q35 and 10q26 from complete and informative partial reads while retaining complex, mosaic and hybrid structures.

The workflow combines D4Z4-array structure, proximal and distal SNV fingerprints, D4F104S1 and pLAM features, repeat origin, SSLP, PAS, optional phasing, BX repeat composition and allele-specific methylation.

---

## Background

Facioscapulohumeral muscular dystrophy (FSHD) is associated with genetic and epigenetic changes of the D4Z4 repeat array at chromosome 4q35. Molecular analysis is complicated by the highly homologous D4Z4 region on chromosome 10q26, multiple permissive and non-permissive haplotypes, large tandem repeat arrays, somatic mosaicism and complex structural rearrangements.

The original DUCKS4 workflow was developed to resolve individual long reads across this region and integrates D4Z4 repeat sizing, chromosome/haplotype assignment, methylation, structural variation and FSHD2-associated variant analysis.

DUCKS4-Advanced builds on this concept by reconstructing allele-level read populations.

---

## Installation

Pull docker image:

```text
docker pull ghcr.io/tamara-nano/ducks4-advanced:latest
```
  
## Usage

For running the tool:

```text
docker run --rm -v $(pwd):/data ghcr.io/tamara-nano/ducks4-advanced:latest --input /data/mysample.bam --allele-methyl --variant
```
For showing more infos:
```text
docker run -it --rm -v $(pwd):/data ghcr.io/tamara-nano/ducks4-advanced:latest --help
```

| tags | infos |
|:--|:--|
| --input | provide input-file. Best start with your basecalled SUP bam or fastq/fastq.gz-file. |
| --allele-methyl | Run allele-specific methylation after allele assignment. |
| --variant | Activate full variant calling with Clair3/Sniffles2 on HG38 + T2T phasing for D4Z4-allele assignment. |
| --phased-t2t-only | Run only T2T FSHD-region phasing for D4Z4-allele assignment, without HG38 variant calling. |
| --phased | Input BAM is a haplotagged.bam and contains phasing tags HP/PS from T2T-phasing; use them for allele assignment |
| --threads | Set your amount of threads. Default is 45. |

The output is saved in the folder where the original input file is located.

---


## Concept

DUCKS4-Advanced reconstructs individual D4Z4 alleles from physical long-read structure and empirically derived sequence fingerprints. Phasing is used as supporting evidence only.

Three types of information are kept separate:

### Structural allele

A structural allele is defined from direct long-read evidence, including repeat number, proximal and distal anchors, D4F104S1, pLAM/PAS, terminal S/L state and D4Z4 repeat composition. Complete reads can define an exact structure, while partial reads provide conservative lower-bound information.

### Sequence fingerprint

Recurrent SNVs and small indels observed within the sample are used to define proximal (`FP`) and distal (`DFP`) sequence backgrounds. Complete reads can directly connect an FP and DFP into a fingerprint component (`FC`).

`DFP-Axx` represents the 4qA/10qA-compatible distal class, whereas `DFP-Bxx` represents the 4qB-compatible distal class.

A fingerprint describes a sequence background and is not identical to a structural allele. Different D4Z4 structures can therefore share the same fingerprint component.

### Phasing

T2T and homolog-specific PS/HP information is used to support linkage of unresolved reads and as an additional QC layer. Phase does not independently define an allele and does not override stronger structural or fingerprint evidence.


---

## Hierarchy for allele reconstruction

Following evidence hierarchy is used for allele reconstruction:

1. Physical D4Z4 structure
2. Proximal and distal sequence fingerprints
3. Complete-read FP <-> DFP linkage
4. Additional locus-specific sequence features
5. T2T and homolog-specific phasing

Lower evidence layers do not override contradictions established by stronger physical or empirical evidence.

Phasing is used as supporting evidence and does not override stronger structural or fingerprint evidence. Opposite HP labels are therefore not sufficient to split one structural allele, while shared HP labels are not sufficient to merge distinct structural or sequence populations. Complete-read structures remain protected from reassignment or mosaic demotion based on phase alone. For unresolved partial reads, phase can support assignment to an already established allele when the structural and fingerprint evidence is compatible. If the available evidence is insufficient, the homolog remains unresolved.

---

## Workflow

### 1. T2T mapping and structural annotation

Long reads are aligned to T2T-CHM13v2.0 and reads overlapping the D4Z4-associated regions on 4q35 and 10q26 are retained. BLAST-based annotation identifies D4F104S1, D4Z4 repeat units, pLAM/PAS and other locus-specific features.

### 2. Haplotype annotation

D4F104S1 SNVs, SSLP, pLAM/PAS, repeat composition and chromosome-specific markers are used to determine compatible chromosome and haplotype classes and if possible exact haplotypes.

#### SSLP annotation
The SSLP is reported using the compact annotation described for the classical D4Z4 haplotypes. Because the T2T reference represents the locus in its full genomic sequence context, DUCKS4 extracts the corresponding SSLP sequence from the aligned reads and converts it to a short annotation for comparison with the published haplotype nomenclature. The SSLP repeat structure and the characteristic ±8 bp variant are retained in this representation. SSLP information is evaluated together with the D4F104S1 SNV pattern for haplotype matching.

### 3. Sample-derived sequence fingerprinting

DUCKS4-Advanced uses a reference-guided fingerprinting approach to distinguish recurrent sequence backgrounds within a sample. Proximal and distal informative reads are realigned to dedicated local references, and recurrent SNVs and small indels are identified directly from the aligned reads. Variants within the repetitive D4Z4 array are only used when the read is anchored to the corresponding proximal or distal region.

Proximal sequence populations are labelled `FP01`, `FP02`, ... and distal populations `DFP-A01`, `DFP-B01`, ... (A/B for qA and qB haplotypes). Complete molecules can directly link proximal and distal fingerprints into fingerprint components (`FC`).

### 4. Allele reconstruction

Final alleles are reconstructed from complete and informative partial reads using physical structure, FP/DFP/FC background and compatible locus-specific sequence information.
Complete reads can define exact repeat numbers. If no molecule spans the complete array, the observed structure is reported as a lower bound (`grN` -> `greater than N RUs`).

### 5. Phasing and unresolved reads

T2T PS/HP information can support linkage of partial or unresolved reads to established allele backgrounds. For unresolved informative populations, dedicated 4qA, 4qB or 10qA realignment and Clair3/LongPhase phasing can be used as an additional rescue and QC layer. Phasing does not independently create, split or merge alleles and does not override stronger structural or fingerprint evidence.

#### Phasing
Reference-based phasing within D4Z4 is complicated by the repetitive structure of the array, high chr4/chr10 sequence similarity and incomplete read coverage. Therefore, T2T and homolog-specific PS/HP information is used as supporting linkage and QC evidence rather than as the primary definition of an allele. If phase infos disagree with stronger structural or fingerprint evidence, the structural/fingerprint assignment is retained and the disagreement is reported for review.

### 6. Mosaic and complex structures

Recurrent complete-read structures are retained separately when supported by physical evidence, even when they share the same fingerprint or phase background. Complex rearrangements that cannot be resolved automatically remain available for manual review.

### 7. Allele-specific methylation and BX repeat composition

Final allele-specific read sets can be realigned to allele-specific references for methylation analysis reporting per-repeat and regional methylation. The BX analysis additionally describes individual D4Z4 repeat units according to BinI/XapI restriction-site patterns.


### Why phasing is used as supporting evidence

Reference-based phasing within D4Z4 is complicated by the repetitive structure of the array, the high sequence similarity between chr4 and chr10 and incomplete coverage of individual alleles. PS/HP information is therefore used as supporting linkage and QC evidence rather than as the primary definition of an allele. During validation, structurally and empirically consistent read populations were occasionally split across different HP groups, while distinct sequence or structural populations could also share the same HP. Proximal and distal phase groups were not always directly linkable when no complete molecule connected both regions, and mosaic or complex structures could share the same phase background as another structural allele. Therefore, allele identity is primarily derived from physical read structure, proximal and distal fingerprints and complete-read FP <-> DFP linkage. T2T and homolog-specific phasing are used to support unresolved assignments and for QC. If phase disagrees with stronger structural or fingerprint evidence, the original allele reconstruction is retained and the discrepancy is reported under `data/assignment-qc/`.

---


## Output


```text
FSHD-analysis_SAMPLE/
├── allele_summary.tsv
├── SAMPLE_FSHD_report.csv
├── SAMPLE_FSHD_report.txt
│
├── haplotypes/
│   ├── original haplotype/read tables
│   ├── read IDs
│   └── haplotypes_complete/all.BAM / BAI
│
├── assigned-alleles/
│   ├── allele_summary.tsv
│   ├── allele_anchors.csv
│   ├── assigned_alleles.csv
│   ├── allele-bam-tables/
│       -> alleles aligned to the T2T-chm13v2.0 reference
│   ├── read-IDs/
│   └── allele_methylation/
│       -> alleles aligned to custom references (id2bam2meth) for improved methylation calling with modkit
│
└── data/
    ├── fingerprint-backgrounds/
    ├── haplotype_prediction/
    ├── assignment-qc/
    ├── phase QC/ 
    └── other intermediate reproducibility files
```

### Main final outputs

| Output | Description |
|:--|:--|
| `allele_summary.tsv` | Concise final allele-level result table |
| `allele_anchors.csv` | Anchor read and merged allele-level evidence |
| `assigned_alleles.csv` | Per-read final allele assignment and assignment evidence |
| `allele-bam-tables/` | Per-allele read tables and BAM/BAI files aligned to T2T-chm13v2.0 |
| `read-IDs/` | Final read-ID lists per allele |
| `allele_methylation/` | Allele-specific methylation results aligned to custom/allele-specific references. |
| `SAMPLE_FSHD_report.html` | final report |
| `SAMPLE_FSHD_report.pdf` | PDF report |

Intermediate BLAST, fingerprinting, phasing, QC and methylation files are retained under `data/`.


---

## ID2bam2meth - analysis of individual read-subsets

The DUCKS4 `ID2bam2meth` utility remains useful for targeted read subsets, custom-reference alignment and detailed methylation analysis.
For the advanced workflow, the preferred read subsets are the final allele-specific read IDs generated under `assigned-alleles/` and `allele-bam-tables/`.

With this script it is possible to define a custom reference from a chosen read via read-id (e.g. choose a complete read from the 4qA output).
Optionally provide a subset of reads to align against (either read-ids via TXT and/or a BAM-file) and optionally call methylation.
For the custom reference the blast-results of this read are annotated within an annotation.bed file and if --methyl is chosen the average methylation will be calculated for each entry within the annotation.bed file.
The results can then be further inspected in a genome viewer like the IGV-browser.

Featuers:
- build a custom reference from a selected read
- realign an allele-specific read subset
- generate a curated BLAST-based annotation BED
- calculate methylation per annotated region and repeat unit
- generate methylation summaries
- run the BX restriction-site analysis

### Mode A: for creating a custom reference
```
docker run -it --rm -v "$(pwd)":/data ghcr.io/tamara-nano/ducks4-advanced:latest id2bam2meth \
  --id_ref READ_ID \
  --bam_ref /data/sample-containing_id_ref-read.bam \
  --bam /data/sample.bam \
  --txt /data/read-ids.txt \
  --methyl
```

**Note**: \
From the read-id a reference FASTA, FAIDX and annotation.bed file from the blast-output is created. \
Please be aware that the id_ref needs to be present in the bam_ref input. \
Methylation is called over all regions from the annotation.bed if no other regions are given (e.g. --regions_bed). \
Therefore a methylation-gradient over all D4Z4-RUs can be called and will be provided as .bedgraph output and for convenience as .bed file with values as labels.
Also an aggregated region as distal_unit containing the most distal repeat and pLAM region is defined and methylation called for it.
The BX restriction site check is automatically run after methylation analysis and results are written to `{out_prefix}/D4Z4_BX_check/`.

### Mode B: providing an existing reference
```
docker run -it --rm -v "$(pwd)":/data ghcr.io/tamara-nano/ducks4-advanced:latest id2bam2meth \
  --ref /data/ref.fasta \
  --bam /data/sample.bam \
  --txt /data/read-ids.txt \
  --methyl \
...
  optional:
...
  --region chr:start-end
```
or
```
  --regions_bed /data/regions.bed
```

**Note**: \
If no custom regions are provided, ID2bam2meth automatically generates annotations from the supplied reference FASTA using BLAST. \
The reference FASTA is automatically copied into the output folder for use with the BX check and IGV visualization.

### Creating the read-ID.txt:
Simply copy the read-IDs you want to subset and filter from the DUCKS4-output tables into a txt-file:

Format read-id.txt:

```
read-id1
read-id3
read-id5
...
```

### For showing more infos:

`docker run -it --rm -v "$(pwd)":/data ghcr.io/tamara-nano/ducks4-advanced:latest id2bam2meth --help`

| **tags** | **Infos** |
|:-----------------------------|:-----------------------------------------|
| **Mode A - create custom ref** | |
| --id_ref | optional, Input read_id from a read (best a complete read) to use as the custom reference (Mode A). |
| --bam_ref | optional, Input BAM file where the reference-read is located (Mode A). |
| --out_prefix | optional, Prefix for naming the reference (default: read-ID) (Mode A). |
| **Mode B - existing ref** | |
| --ref | optional, Use an existing reference FASTA (Mode B). |
| **reads for alignment** | |
| --bam | optional, Input BAM to extract/align reads against the reference. |
| --txt | optional, Optional read_id.txt to subset reads from --bam to be aligned to reference. |
| **Methylation** | |
| --methyl | optional, Methylation calculation for the reads. Methylation calculation happens over the whole repeat-array for the mean CpG value for each D4Z4. |
| --region | optional, single region in chr:start-end format. |
| --regions_bed | optional, BED file with multiple regions. |
| --skip-bx | optional, skip the BX restriction site check. |
| **Outputs** | |
| --threads | optional, Set your amount of threads. Default is 45. |
| --out_path | optional, Give output path, default: path from --bam_ref. |

### Output:

- Mode A: reference.fasta, reference.fasta.fai, annotation.bed generated from read_id
- Mode A & B: The annotation is generated automatically using BLAST and curate_annotation_bed.py.
- aligned reads.bam / subset-reads.bam to reference
- Methylation: alignedreads.bedgraph, alignedreads.bed, alignedreads.methylbed, modkit-stats.tsv for the annotated regions if no region/regions_bed is provided.
- D4Z4_BX_check/: BinI/XapI restriction site classification per RU (see BX check section above)


---

## Publication

The original DUCKS4 workflow is described in:

<Löwenstern T, Madritsch S, Horner D, Brait N, Güleray Lafci N, Schachner A, Gerykova Bujalkova M, Kałużewski T, Szyld P, Hengstschläger M, Dremsek P, Laccone F. DUCKS4: a comprehensive workflow for Nanopore sequencing analysis of facioscapulohumeral muscular dystrophy (FSHD). Hum Genomics. 2026 Feb 6;20(1):48. doi: 10.1186/s40246-026-00921-2. PMID: 41652446; PMCID: PMC12977661.>

If DUCKS4-Advanced is used for research, please cite the original DUCKS4 publication and specify that the experimental advanced allele-reconstruction branch was used.

---

## License / disclaimer

DUCKS4-Advanced is provided for research and validation purposes. It is not intended to replace independent clinical validation or expert review.
