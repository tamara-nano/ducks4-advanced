
# DUCKS4-Advanced

Experimental major update of DUCKS4 introducing advanced allele assignment and based on de-novo variant fingerprinting and homolog-specific phasing. 

> 
> **ATTENTION: This is an EXPERIMENTAL repository and any output needs to be carefully revised.**  
> 

DUCKS4-Advanced extends the original **DUCKS4** workflow with automated reconstruction of individual D4Z4 alleles from long-read Oxford Nanopore data.

The main aim is to resolve the four D4Z4 chromosome backgrounds at **4q35** and **10q26** from complete and informative partial reads while retaining complex, mosaic and hybrid structures.

The workflow combines physical D4Z4 structure, empirical proximal and distal SNV fingerprints, D4F104S1 and pLAM features, repeat origin, SSLP, PAS, optional phasing, BX repeat composition and allele-specific methylation.

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

or

Build image with dockerfile:

download repository and unzip it

```text
cd /path/ducks4/

docker build -t ghcr.io/tamara-nano/ducks4-advanced:latest .
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

## Core concept

The central rule of DUCKS4-Advanced is:

Physical D4Z4 structure and empirical sequence fingerprints define biological alleles whereas phasing is supportive evidence only.

Three concepts are therefore kept separate:

### 1. Structural D4Z4 allele

Defined from physical long-read evidence such as:

- repeat number
- complete proximal-to-distal structure
- D4F104S1 state
- pLAM / PAS
- S/L terminal unit
- D4Z4 repeat composition
- complete versus partial read status

### 2. Empirical sequence background

Defined from SNVs observed directly in the sequenced cohort:

- proximal fingerprint: `FP01`, `FP02`, ...
- distal fingerprint: `DFP-A01`, `DFP-B01`, ...
- FP ↔ DFP linkage from complete reads
- empirical component: `FC01`, `FC02`, ...

An empirical fingerprint is defined by recurrent sequence variants like SNVs and INDELS. Different structural alleles may occur on the same fingerprint background, for example when a mosaic structural derivative is present.

### 3. Phase group

LongPhase PS/HP information describes a phasing solution, but does not independently define a D4Z4 allele.

This distinction is important because local phasing within D4Z4 can:

- split reads from one structural/fingerprint population into HP1 and HP2;
- place strongly different SNV populations into the same phase group;
- mix reads originating from the paralogous chr4 and chr10 regions;
- assign a mosaic allele to the same inherited haplotype background as its parental allele.

For this reason LongPhase is used for linkage, rescue and QC but it will not override stronger structural or fingerprint evidence.

---

## Evidence hierarchy

DUCKS4-Advanced uses the following hierarchy for allele reconstruction:

1. **Complete physical D4Z4 structure**
2. **Empirical proximal and distal SNV fingerprints**
3. **Complete-read FP ↔ DFP bridge**
4. **From the fingerprints derived Empirical components**
5. **D4F104S1 / SSLP / PAS / other sequence features**
6. **Generic T2T phase for supporting linkage**
7. **Dedicated homolog-specific LongPhase for rescue / QC**

A lower evidence layer must not overwrite a contradiction established by a higher layer.

### Consequences

- Opposite HP labels alone do not justify splitting one structural allele into two.
- Shared HP labels do not justify merging structurally or sequence-distinct populations.
- Established complete reads are not reassigned because of phase.
- A strongly supported complete structure is not demoted to mosaic solely because it shares a phase background with another allele.
- Phase may help an unresolved partial read follow an already established allele if structure and fingerprint evidence remain compatible.
- If the data do not resolve a homolog, DUCKS4-Advanced keeps it unresolved.

---

## Workflow

### 1. Input and T2T mapping

The workflow starts from basecalled long-read sequencing data and aligns reads to **T2T-CHM13v2.0**.

Reads overlapping the FSHD-associated regions on 4q35 and 10q26 are retained for downstream analysis.

### 2. Structural annotation

D4Z4-associated elements are identified by BLAST-based annotation.

The orientation used throughout DUCKS4 is:

```text
proximal                                               distal
D4F104S1  →  D4Z4 repeat array  →  terminal D4Z4  →  pLAM / PAS
```

The workflow evaluates:

- **D4F104S1** as the proximal anchor
- individual D4Z4 repeat units
- **pLAM** as the distal anchor
- PAS sequence
- SSLP structure
- S/L terminal repeat state
- chr4/chr10-specific structural markers
- D4Z4 repeat origin

Repeat origin is evaluated independently for individual D4Z4 units using local chr4/chr10 sequence similarity.

### 3. Haplotype prediction

D4F104S1 SNVs, SSLP structure, pLAM, PAS, D4Z4 composition and chromosome-specific markers are combined to predict chromosome and broad haplotype class.

Automated subtype calls remain conservative when two reference haplotypes are sequence-indistinguishable using the available markers. Such cases remain unresolved until an independent/manual discriminator is available.

### 4. Empirical proximal and distal fingerprinting

DUCKS4-Advanced uses an "reference-guided but empirically learned fingerprinting approach" to distinguish sequence backgrounds at the D4Z4 locus. Proximal and distal informative reads are remapped to dedicated local references that provide a common coordinate system for variant comparison. Recurrent SNVs and small indels are then identified directly from the sample and combined into sparse per-read sequence fingerprints. Proximal fingerprint populations (`FP`) and distal fingerprint populations (`DFP`) are resolved independently, with repeat-derived variants accepted only when supported by the appropriate physical anchor such as D4F104S1 or pLAM. Complete molecules can directly link proximal and distal fingerprints into empirical components (`FC`). These fingerprints describe recurrent sequence backgrounds and different D4Z4 structures may therefore share the same fingerprint background. T2T or homolog-specific phasing is used as supporting linkage and QC information but are not used as primary basis for allele reconstruction.

Allele-informative reads are used to discover empirical SNV patterns.

Proximal populations are assigned identifiers such as:

```text
FP01
FP02
FP03
...
```

Distal populations are assigned identifiers such as:

```text
DFP-A01
DFP-A02
DFP-B01
...
```

### 5. Empirical component reconstruction

FP and DFP populations are connected into empirical components:

```text
FC01
FC02
FC03
...
```

A complete-read FP <-> DFP bridge provides the strongest local linkage.
When no complete bridge exists, additional evidence may be used conservatively, but an empirical component is not allowed to erase an independently supported structural difference.

### 6. Structural allele reconstruction

Final alleles are built from complete and informative partial reads.

The workflow integrates:

- exact or lower-bound RU count
- proximal and distal anchors
- D4F104S1 state
- pLAM / PAS
- S/L state
- repeat origin
- empirical FP / DFP / FC background
- compatible haplotype information
- structural integrity
- supporting read counts

#### SSLP annotation

The SSLP is reported using the compact annotation described for the classical D4Z4 haplotypes. Because the T2T reference represents the locus in its full genomic sequence context, DUCKS4 extracts the corresponding SSLP sequence from the aligned reads and converts it to a standardized short annotation for comparison with the published haplotype nomenclature. The SSLP repeat structure and the characteristic ±8 bp variant are retained in this representation. SSLP information is evaluated together with the D4F104S1 SNV pattern for haplotype matching.

The complete observed SSLP sequence is retained in `SSLP.txt` in the `sslp_sequence_manual` column. The full compressed repeat annotation is available as `sslp_annotation`, while the shortened haplotype-style representation used for matching is stored as `sslp_haplotype_annotation`.

#### Complete alleles

A complete/spanning read can define an exact D4Z4 repeat length.

#### Partial alleles

A partial read provides a lower bound and the allele-size is based on the longest compatible observed proximal or distal read.

Partial alleles are therefore reported as:

```text
grN      -> "greater than N RUs"
```

### 7. Mosaic and complex structures

A recurrent structural population that differs from the dominant allele is retained separately when supported by complete-read evidence.
Such a mosaic or complex structure can share the same empirical fingerprint (FP) background and the same phase background as another structural allele. Shared fingerprint or phase therefore indicates a common sequence background, but does not determine whether two structural populations represent constitutional alleles, a mosaic derivative, or another complex configuration.

Small minor populations can be retained as mosaic candidates, while strongly supported complete populations are protected from automatic mosaic demotion.

### 8. T2T phasing

T2T PS/HP information is used as supporting evidence for:

- chromosome-background linkage
- follower confirmation
- subtelomeric linkage
- technical QC

It is intentionally weighted weaker for allele-assignment than physical structure and empirical fingerprints.

### 9. homolog-specific phasing

For unresolved informative reads, DUCKS4-Advanced can optionally remap and phase against dedicated:

- 4qA
- 4qB
- 10qA

references using Clair3 + LongPhase.

Dedicated phase may therefore support assignment of an unresolved partial read to an existing compatible allele and/or provide PS/HP QC information.

Dedicated phase may **not**:

- create a new biological allele solely from HP1/HP2;
- split an established structural/fingerprint population;
- merge different SNV populations because they share an HP;
- reassign established complete reads because of phase;
- demote a strongly supported complete structure to mosaic solely because of phase.

Raw dedicated phase groups can be retained as **phase-shadow QC views** for manual inspection in IGV.


#### Why phasing is not used as primary allele evidence

Phasing is used in DUCKS4-Advanced as supporting evidence, particularly in unique subtelomeric regions and for assigning otherwise unresolved reads to established alleles. However, due to the highly repetitive nature of D4Z4 and the strong sequence similarity between chr4 and chr10, reference-based phasing is not always reliable for reconstructing individual D4Z4 alleles. Informative SNVs are unevenly distributed, and alleles that differ substantially from the reference, for example through contractions, expansions or complex rearrangements, may show fragmented alignments that do not necessarily reflect their actual structure. Furthermore, many reads cover only proximal or distal parts of the array, meaning that shared phase information does not provide direct evidence that these regions belong to the same allele. This is particularly relevant for independently phased proximal and distal regions, where PS/HP labels are local and cannot be assumed to correspond across separate analyses without additional evidence. Structural mosaic alleles may also share the same inherited phase background, making phasing alone insufficient to distinguish different structural populations. For these reasons, DUCKS4-Advanced primarily relies on observed read structures, empirical proximal and distal sequence fingerprints, and direct connections established by complete molecules, while phasing is used to support or refine allele assignments where appropriate.

#### Problems observed during validation

During the development and validation of DUCKS4-Advanced, several limitations of phase-based allele assignment were observed. Structurally and empirically consistent allele populations were occasionally separated into different HP groups, while distinct SNV or structural populations could also be assigned to the same HP. Similar discrepancies were observed with dedicated-reference phasing, where reads belonging to the same proximal/distal fingerprint component could show different local phase assignments. In addition, proximal and distal phase information could not always be directly connected, particularly when no complete reads were available to bridge both regions. Structural mosaic populations presented another limitation, as they could share the same inherited phase background as their parental allele and therefore remain indistinguishable by phasing alone. Together with the high sequence similarity between chr4 and chr10 and the fragmented alignments frequently observed within the D4Z4 region, these observations showed that phase-based reconstruction could result in both artificial separation of individual alleles and merging of biologically distinct structural populations.

Consequently, DUCKS4-Advanced establishes allele identity primarily through observed read structures, empirical proximal and distal sequence fingerprints, and direct FP <-> DFP connections supported by complete reads. T2T phasing is subsequently used as supporting evidence for chromosome or subtelomeric linkage, while dedicated 4qA/4qB/10qA phasing is applied as conditional rescue or for additional quality control of unresolved populations. If phase assignments disagree with stronger structural or empirical evidence, the original allele assignment is retained and the discrepancy is documented rather than forcing a phase-based reassignment. Detailed allele assignment and phasing results are available under `data/assignment-qc/`.


### 10. Allele-specific methylation

Allele-specific BAMs can be used for methylation calling and summarized across defined D4Z4 regions and is reported for each array-element.

Reported regions include, where available:

- proximal anchor
- proximal repeat unit
- array core
- internal repeats
- distal repeat
- pLAM
- distal unit
- distal DR1-region on the last full-size repeat-unit

### 11. BX repeat composition

The DUCKS4 BX-check classifies individual D4Z4 repeat units using BinI/XapI restriction-site patterns and generates a per-RU array structure.

This provides an additional structural description of chr4-like, chr10-like and hybrid repeat units and is particularly useful for complex arrays.

---


## Output


```text
FSHD-analysis_SAMPLE/
├── allele_summary.tsv
├── SAMPLE_FSHD_report.html
├── SAMPLE_FSHD_report.pdf
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

> Löwenstern T, Madritsch S, Horner D, Brait N, Güleray Lafci N, Schachner A, Gerykova Bujalkova M, Kałużewski T, Szyld P, Hengstschläger M, Dremsek P, Laccone F.  
> **DUCKS4: a comprehensive workflow for Nanopore sequencing analysis of facioscapulohumeral muscular dystrophy (FSHD).**  
> *Human Genomics*. 2026;20(1):48.  
> doi: **10.1186/s40246-026-00921-2**

If DUCKS4-Advanced is used for research, please cite the original DUCKS4 publication and specify that the experimental advanced allele-reconstruction branch was used.

---

## License / disclaimer

DUCKS4-Advanced is provided for research and validation purposes. It is not intended to replace independent clinical validation or expert review.
