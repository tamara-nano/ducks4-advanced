
# DUCKS4-Advanced

Experimental major update of DUCKS4 introducing advanced allele assignment and based on de-novo variant fingerprinting and homolog-specific phasing. 

> 
> **ATTENTION: This is an EXPERIMENTAL repository and any output needs to be carefully revised.**  
> 

DUCKS4-Advanced extends the original **DUCKS4** workflow with automated reconstruction of individual D4Z4 alleles from long-read Oxford Nanopore data.

The main aim is to resolve the four D4Z4 chromosome backgrounds at **4q35** and **10q26** from complete and informative partial reads while retaining complex, mosaic and hybrid structures instead of forcing every sample into a simple diploid model.

The workflow combines physical D4Z4 structure, empirical proximal and distal SNV fingerprints, D4F104S1 and pLAM features, repeat origin, SSLP, PAS, optional phasing, BX repeat composition and allele-specific methylation.


## Installation

Pull docker image:

```text
docker pull ghcr.io/tamara-nano/ducks4-advanced:v0.1.0
...
or

Build image with dockerfile:

download repository and unzip it

```text
cd /path/ducks4/

docker build -t ghcr.io/tamara-nano/ducks4-advanced:v0.1.0 .
...
  
## Usage

For running the tool:

docker run --rm -v $(pwd):/data ghcr.io/tamara-nano/ducks4-advanced:v0.1.0 --input /data/mysample.bam --allele-methyl --variant

For showing more infos:

docker run -it --rm -v $(pwd):/data ghcr.io/tamara-nano/ducks4-advanced:v0.1.0 --help
tags 	Infos
--input 	provide input-file. Best start with your basecalled SUP bam or fastq/fastq.gz-file.
--allele-methyl        automated allele-specific alignment and phasing with id2bam2meth
--phased-t2t-only        phasing is only performed for the 4q35 and 10q26 region for T2T-chm13v2.0 for D4Z4 allele-assignment.
--variant 	optional, alignment to HG38, variant calling with clair3, sniffles2 (sniffles also calls against the T2T-aligned bam-file), phasing with longphase and variant annotation with SNPEff and SnpSift against ClinVar (v20250729).
--methyl 	optional, legacy methylation calling with modkit, target region: chr4:193540172-193543634.
--threads 	optional, set threads.
--version 	show program's version number and exit.

The output is saved in the folder where the original input file is located.

---

## Background

Facioscapulohumeral muscular dystrophy (FSHD) is associated with genetic and epigenetic changes of the D4Z4 repeat array at chromosome 4q35. Molecular analysis is complicated by the highly homologous D4Z4 region on chromosome 10q26, multiple permissive and non-permissive haplotypes, large tandem repeat arrays, somatic mosaicism and complex structural rearrangements.

The original DUCKS4 workflow was developed to resolve individual long reads across this region and integrates D4Z4 repeat sizing, chromosome/haplotype assignment, methylation, structural variation and FSHD2-associated variant analysis.

DUCKS4-Advanced builds on this concept by reconstructing allele-level read populations.

---

## Core concept

The central rule of DUCKS4-Advanced is:

Physical D4Z4 structure and empirical sequence fingerprints define biological alleles. Phasing is supportive evidence only.

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

An empirical fingerprint represents an inherited sequence background. Different structural alleles may occur on the same fingerprint background, for example when a mosaic structural derivative is present.

### 3. Phase group

LongPhase PS/HP information describes a phasing solution, but does not independently define a D4Z4 allele.

This distinction is important because local phasing within D4Z4 can:

- split reads from one structural/fingerprint population into HP1 and HP2;
- place strongly different SNV populations into the same phase group;
- mix reads originating from the paralogous chr4 and chr10 regions;
- assign a mosaic allele to the same inherited haplotype background as its parental allele.

For this reason, LongPhase is used for linkage, rescue and QC, but must not override stronger structural or fingerprint evidence.

---

## Evidence hierarchy

DUCKS4-Advanced uses the following hierarchy for allele reconstruction:

1. **Complete physical D4Z4 structure**
2. **Empirical proximal and distal SNV fingerprints**
3. **Complete-read FP ↔ DFP bridge**
4. **Empirical component**
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

The workflow starts from basecalled long-read sequencing data and maps reads to **T2T-CHM13v2.0**.

Reads overlapping the FSHD-associated regions on chr4q35 and chr10q26 are retained for downstream analysis.

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

Allele-informative reads are used to discover empirical SNV patterns without forcing a predefined diploid model.

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

Complete reads spanning both informative ends provide direct physical links between proximal and distal fingerprint populations.

### 5. Empirical component reconstruction

FP and DFP populations are connected into empirical components:

```text
FC01
FC02
FC03
...
```

A complete-read FP ↔ DFP bridge provides the strongest local linkage.

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

#### Complete alleles

A complete/spanning read can define an exact D4Z4 repeat length.

#### Partial alleles

A partial read provides a lower bound, not an exact allele size.

Partial alleles are therefore reported as:

```text
grN      -> "greater N"
```

meaning **at least N repeat units**.

The lower bound is based on the longest compatible observed proximal or distal structure and is never calculated by summing two non-overlapping partial reads.

### 7. Mosaic and complex structures

A recurrent structural population that differs from the dominant allele is retained separately when supported by complete-read evidence.

A mosaic/complex structure can share the same inherited FP background and the same phase background as its parental allele.

Phase therefore provides possible parental linkage, but does not decide whether a structural population is constitutional or mosaic.

Small minor populations can be retained as mosaic candidates; strongly supported complete populations are protected from automatic mosaic demotion.

### 8. Generic T2T phase

Generic T2T PS/HP information is used as supporting evidence for:

- chromosome-background linkage
- follower confirmation
- subtelomeric linkage
- technical QC

It is intentionally weaker than physical structure and empirical fingerprints.

### 9. Dedicated homolog-specific rescue phase

For unresolved informative reads, DUCKS4-Advanced can optionally remap and phase against dedicated:

- 4qA
- 4qB
- 10qA

references using Clair3 + LongPhase.

Dedicated phase may:

- support assignment of an unresolved partial read to an existing compatible allele;
- provide PS/HP QC information;

Dedicated phase may **not**:

- create a new biological allele solely from HP1/HP2;
- split an established structural/fingerprint population;
- merge different SNV populations because they share an HP;
- reassign established complete reads because of phase;
- demote a strongly supported complete structure to mosaic solely because of phase.

Raw dedicated phase groups can be retained as **phase-shadow QC views** for manual inspection in IGV.

#### Why phasing is not used as primary allele evidence

Phasing remains useful in DUCKS4-Advanced, especially in unique subtelomeric sequence and for linking unresolved reads to an already established allele. The limitation is specific to the D4Z4 locus: a PS/HP group is a local computational phasing solution and does not necessarily correspond one-to-one with a biological D4Z4 allele.

Several properties of the locus make direct phase-based allele reconstruction unreliable:

- **D4Z4 is highly repetitive and chr4/chr10 are strongly paralogous.** Reads from both loci can align to very similar sequence, and the informative SNVs available for phasing are unevenly distributed. Phase within the repeat region is therefore more vulnerable to ambiguous or locally fragmented alignments than phase in unique subtelomeric sequence.
- **The patient allele can differ strongly from the reference array.** Contracted, expanded or otherwise structurally different arrays may align in several pieces against the reference. Such mapping fragmentation is not by itself evidence for a second allele or a structural rearrangement.
- **Many informative reads are partial.** A proximal read and a distal read may each receive a phase label without physically connecting both ends of the allele. Phase can support a proposed linkage, but it cannot replace a complete molecule that directly bridges proximal and distal sequence.
- **Dedicated reference phasing is local.** The 4qA proximal and distal rescue analyses are separate phasing problems. Their PS/HP labels are therefore treated as local labels until they are connected by shared complete molecules. `HP1` in one local analysis is not assumed to be equivalent to `HP1` in another independent analysis.
- **A structural mosaic can share the same inherited phase background as its parent allele.** Phase can therefore support parental linkage but cannot decide whether two different complete structures represent the same allele, a constitutional second allele or a mosaic derivative.

#### Problems observed during validation

These limitations were not only theoretical. During development, several recurrent failure modes were observed:

- one structurally and empirically coherent allele population could be split across both HP1 and HP2;
- different SNV or structural populations could be placed into the same HP;
- dedicated-reference phasing could show a local HP mixture within an otherwise concordant FP/DFP component;
- proximal and distal phase assignments could not always be connected directly because no complete molecule covered both informative regions;
- phase alone could not distinguish a structural mosaic from its constitutional parent when both shared the same chromosome background;
- chr4/chr10 paralogy and repeat-associated mapping fragmentation could create phase patterns that were technically reproducible but biologically misleading if interpreted as allele identity.

Using PS/HP as the primary allele-definition layer would therefore cause both over-splitting and over-merging: one true allele can appear as two phase populations, while two genuinely different structural populations can appear on the same phase background.

For this reason, DUCKS4-Advanced uses phase only after the main allele evidence has been established. Complete physical structure, empirical proximal/distal fingerprints and complete-read FP ↔ DFP bridges remain primary. Generic T2T phase is used for supportive chromosome/subtelomere linkage and follower confirmation. Dedicated 4qA/4qB/10qA phasing is run only as conditional rescue or QC for unresolved populations.

If phase disagrees with stronger structural or empirical evidence, the allele assignment is retained and the phase disagreement is reported as QC.

Detailed assignment and phasing audits are retained under `data/assignment-qc/`.

### 10. Allele-specific methylation

Allele-specific BAMs can be used for methylation calling and summarized across defined D4Z4 regions.

Reported regions include, where available:

- proximal anchor
- proximal repeat unit
- array core
- internal repeats
- distal repeat
- pLAM
- distal unit

The distal DR1 region is retained as an important FSHD-oriented methylation marker.

### 11. BX repeat composition

The DUCKS4 BX check classifies individual D4Z4 repeat units using BinI/XapI restriction-site patterns and generates a per-RU array structure.

This provides an additional structural description of chr4-like, chr10-like and hybrid repeat units and is particularly useful for complex arrays.

BX information is treated as structural characterization and is reported independently from LongPhase.

---

## Method characteristics and current limitations

DUCKS4-Advanced uses an evidence hierarchy tailored to Oxford Nanopore long-read data and the D4Z4 locus. The workflow combines **physical molecule structure** with sample-specific sequence backgrounds and native methylation information.

### Features

- **Direct structural evidence from very long molecules.** Complete reads can physically connect D4F104S1, the D4Z4 array and the distal pLAM region. When such a molecule is available, its observed structure has priority over inferred linkage or phase evidence.
- **Complete and partial reads are both retained.** Long alleles do not need to be discarded when no complete molecule is available. Proximal and distal observations are kept independently and reported conservatively as lower-bound structures.
- **Empirical fingerprinting is de novo.** Proximal and distal FP/DFP populations are learned from recurrent SNVs and small indels in the sample and are not forced into a predefined number of haplotypes.
- **Physical FP ↔ DFP linkage.** Complete reads can directly connect independently discovered proximal and distal sequence backgrounds into an empirical component.
- **Structure and inherited background are kept separate.** Different exact D4Z4 structures can occur on the same FP/DFP background. This is essential for mosaic and other structurally derived alleles.
- **Mosaic/complex populations are not automatically collapsed.** A recurrent minor complete-read structure can remain separate even when it shares fingerprint and phase background with a constitutional parent allele.
- **Native methylation can be retained allele-specifically.** Final allele BAMs and custom references allow methylation to be evaluated across individual repeat units and defined proximal/distal regions.
- **Uncertainty is retained.** Reads, homologs or partial structures that cannot be assigned safely remain unresolved rather than being forced into a four-allele or HP1/HP2 model.
- **Detailed evidence remains inspectable.** Per-allele BAMs, read lists, fingerprint tables, assignment audits and phase-shadow outputs are retained for manual review and reproducibility.

### Current limitations and difficult cases

- **ONT sequence errors require population-level evidence.** Single-read SNV or indel differences are not sufficient to define a fingerprint. Recurrent population consensuses are required, and indel-only splits are intentionally more conservative.
- **D4Z4 repeat mapping remains difficult.** Internal repeat units can map interchangeably to similar reference units. Proximal repeat-derived fingerprint calls are therefore accepted only with physical proximal anchoring, and distal repeat-derived calls require physical distal/pLAM anchoring.
- **Sparse coverage can leave fingerprints unresolved.** A true sequence background may not receive an FP or DFP when too few reads cover discriminating sites. Missing fingerprint evidence is not interpreted as evidence that two alleles are identical.
- **Fingerprint IDs are sample-specific.** `FP01`, `DFP-A01` and `FC01` are local identifiers and are not intended to represent universal haplotypes across samples.
- **A fingerprint component is not identical to a structural allele.** Mosaic or other derived structures can share the same inherited component. Complete physical structures therefore remain separate when they differ.
- **Very long arrays may remain partial.** If no molecule physically spans the whole array, DUCKS4-Advanced reports a lower bound.
- **Generic and dedicated phasing can be locally inconsistent.** Phase is therefore supporting evidence only; the known problems and safeguards are described above.
- **Low-level mosaic detection depends on molecule support.** A minor structure needs positive structural evidence. Spanning-read counts are affected by read length and are not used directly as an unbiased estimate of mosaic fraction.
- **Complex rearrangements can exceed the current automatic model.** Duplicated arrays, large proximal deletions and unusual breakpoint structures may be retained for structural review and still require manual BAM/IGV inspection.
- **Translocation calls are deliberately conservative.** Internal hybrid/BX repeat composition alone does not define a translocation. A possible chr4/chr10 translocation is considered only when proximal chromosome origin and distal haplotype origin disagree (`chr4 → chr10` or `chr10 → chr4`) and the combination is not explained by a known haplotype.

Interpretation is most robust when direct long-molecule structure and recurrent empirical sequence evidence support the same result. The main remaining difficulty is the reconstruction of very long, low-coverage or structurally unusual arrays when no single molecule spans all informative regions.

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
│   └── BAM / BAI
│
├── assigned-alleles/
│   ├── allele_summary.tsv
│   ├── allele_anchors.csv
│   ├── assigned_alleles.csv
│   ├── allele-bam-tables/
        -> alleles aligned to the T2T-chm13v2.0 reference
│   ├── read-IDs/
│   └── allele_methylation/
        --> alleles aligned to custom references (id2bam2meth) for improved methylation calling with modkit
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

## Reported allele properties

Depending on available evidence, the final allele-level report contains:

- chromosome
- broad haplotype class
- predicted haplotype subtype
- exact RU count or lower-bound `grN`
- complete / proximal / distal read support
- D4F104S1 type
- D4F104S1 SNV consensus
- SSLP consensus
- pLAM / PAS
- S/L state
- empirical FP / DFP / FC
- generic and dedicated phase information
- BX array composition
- structural warnings
- mosaic / complex classification
- assignment confidence
- methylation values
- FSHD-oriented interpretation

---

## Interpretation safeguards

DUCKS4-Advanced intentionally avoids overcalling when the available reads cannot resolve a biological distinction.

Examples include:

- sequence-identical haplotype subtypes requiring an independent/manual discriminator;
- partial alleles with only lower-bound repeat length;
- unresolved homologs;
- locally unreliable phasing;
- highly paralogous chr4/chr10 alignments;
- structurally complex arrays;
- low-support mosaic candidates.

The workflow reports these states explicitly instead of forcing a complete diploid solution.

---

## Manual review

Even though the advanced workflow performs automatic allele reconstruction, manual review remains recommended for:

- contracted pathogenic candidates
- complex or hybrid structures
- mosaic candidates
- unresolved homologs
- ambiguous haplotype subtypes
- unusual chr4/chr10 repeat composition
- phase/fingerprint disagreement

Per-allele BAMs, phase-shadow QC BAMs and the final evidence tables are designed to support inspection in IGV.

---

## ID2bam2meth

The original DUCKS4 `ID2bam2meth` utility remains useful for targeted read subsets, custom-reference alignment and detailed methylation analysis.

It can:

- build a custom reference from a selected read;
- realign an allele-specific read subset;
- generate a curated BLAST-based annotation BED;
- calculate methylation per annotated region and repeat unit;
- generate distal-unit methylation summaries;
- run the BX restriction-site analysis.

For the advanced workflow, the preferred read subsets are the final allele-specific read IDs generated under `assigned-alleles/` and `allele-bam-tables/`.

---

## Development status

Current development focuses on:

- fingerprint-first allele reconstruction
- complete and partial allele integration
- mosaic detection
- complex/hybrid allele handling
- phase-safe unresolved-read rescue
- allele-specific methylation
- automatic HTML/PDF reporting

This branch should currently be regarded as a **research and validation workflow**.

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
