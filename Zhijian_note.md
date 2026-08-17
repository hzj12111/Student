# EDGE:QC

How to assign the 5′, 3′ and internal reads, extract the UMI, and remove artificial sequences
# 1. Overall workflow

The simplified workflow of RNAEdgeFlow is:
Raw FASTQ read
    ↓
Identify 3P, 5P, internal or uncertain reads
    ↓
Extract and save the UMI
    ↓
Remove barcode, fixed sequence, primer and adapter
    ↓
Filter reads that are too short
    ↓
Map the remaining biological insert with STAR

The order is important:
Use barcode/fixed sequence for read assignment
    ↓
Extract and save the UMI
    ↓
Remove artificial sequences
    ↓
Map the biological sequence

Barcode and fixed sequences must be examined before trimming because they are used to determine whether a read is a 3P or 5P terminal read.

The UMI must also be extracted before trimming. After it has been saved in the read name or other metadata, it can later be used for deduplication even though it is no longer present in the read sequence.

# 2. How are 3P, 5P, internal and uncertain reads assigned?
RNAEdgeFlow uses the positions of barcode, UMI and fixed sequences defined in the profile to classify reads before genome mapping.
The simplified classification logic is:
Check whether the read matches the strict 3P pattern
    │
    ├── Yes → assign as a 3P read
    │
    └── No
         ↓
Check whether the read matches the strict 5P pattern
    │
    ├── Yes → assign as a 5P read
    │
    └── No
         ↓
Check whether it matches a loose terminal pattern
    │
    ├── Yes → assign as uncertain
    │
    └── No → assign as an internal read

Therefore:
Strict 3P match       → 3P read
Strict 5P match       → 5P read
Loose terminal match  → uncertain read
No terminal match     → internal read
The exact classification depends on the barcode list, fixed sequences, expected positions and mismatch settings in the profile.

2.1 Example 3P structure

A simplified 3P read start is:

6 bp barcode      8 bp UMI       TTT        biological insert
positions 1–6     positions 7–14 positions 15–17

The corresponding profile settings are:

P3_BC_Pattern=XXXXXXNNNNNNNNXXX
P3_UmiRegion=7:14

P3_FixedSeq1=
P3_FixedSeq1_File=./tests/p3_barcode_position1_6.txt
P3_FixedSeq1Region=1:6
P3_FixedSeq1Mismatch=0
P3_FixedSeq1MismatchLoose=

P3_FixedSeq2=TTT
P3_FixedSeq2Region=15:17
P3_FixedSeq2Mismatch=0
P3_FixedSeq2MismatchLoose=

The meanings are:

Positions 1–6
Check whether the sequence belongs to the allowed 3P barcode list.

Positions 7–14
Extract the 8 bp UMI.

Positions 15–17
Check whether the sequence is TTT.

P3_FixedSeq1Mismatch=0 and P3_FixedSeq2Mismatch=0 mean that the strict pattern does not allow mismatches.

The loose mismatch settings can be used to identify reads that resemble terminal reads but do not meet the strict criteria.

Conceptually:

uncertain reads = loose terminal matches − strict terminal matches
2.2 Example 5P structure

A simplified 5P read start is:

AGTC        8 bp UMI       CATCAGGG       biological insert
positions   positions      positions
1–4         5–12           13–20

The corresponding profile settings are:

P5_BC_Pattern=XXXXNNNNNNNNXXXXX
P5_UmiRegion=5:12

P5_FixedSeq1=AGTC
P5_FixedSeq1Region=1:4
P5_FixedSeq1Mismatch=0
P5_FixedSeq1MismatchLoose=

P5_FixedSeq2=CATCAGGG
P5_FixedSeq2Region=13:20
P5_FixedSeq2Mismatch=0
P5_FixedSeq2MismatchLoose=

RNAEdgeFlow checks whether AGTC and CATCAGGG occur at the expected positions.

If both strict fixed-sequence requirements are satisfied, the read can be assigned as a 5P terminal read.

2.3 Main output files

The output of split_terminal may include:

process/*_3P_R1.fastq.gz
process/*_5P_R1.fastq.gz
process/*_internal_R1.fastq.gz

EDGES18R1_split_confidence_stats.tsv
EDGES18R1_split_subcommand_stats.tsv

The FASTQ files contain the reads assigned to different categories.

The statistics files can be used to check:

Number of 3P reads
Number of 5P reads
Number of internal reads
Number of uncertain reads
Confidence of the classification

# 3. A concrete 3P read example
The following sequence is a hypothetical example used to explain the processing steps. It is not a real MicEdges read or a confirmed adapter sequence from the experiment.
Suppose the raw read is:
[ACGTAC][GATTACGA][TTT][CCGGATGCTAACGTAGGCTTACGA][AGATCGGAAGAGC]
The sequence contains:
ACGTAC= 6 bp 3P barcode
GATTACGA= 8 bp UMI
TTT= 3P fixed sequence
CCGGATGCTAACGTAGGCTTACGA= biological insert
AGATCGGAAGAGC= adapter sequence
Without brackets, the raw read is:
ACGTACGATTACGATTTCCGGATGCTAACGTAGGCTTACGAAGATCGGAAGAGC
3.1 Step 1: Assign the read as a 3P read
RNAEdgeFlow first checks the expected positions:
Positions 1–6:    ACGTAC
Positions 7–14:   GATTACGA
Positions 15–17:  TTT

The interpretation is:
ACGTAC
→ matches an allowed barcode in the 3P barcode file

GATTACGA
→ occupies the expected 8 bp UMI region

TTT
→ matches P3_FixedSeq2 at positions 15–17

Because the barcode and fixed sequence occur at the expected positions, this read is assigned as a 3P read.

Conceptually, it enters:

sample_3P_R1.fastq.gz

At this point, the artificial sequences have been used for classification, but they should not be mapped to the genome.

3.2 Step 2: Extract and save the UMI

The UMI region is defined as:

P3_UmiRegion=7:14

Therefore, the extracted UMI is:GATTACGA

Conceptually:
Raw read:
[ACGTAC][GATTACGA][TTT][CCGGATGCTAACGTAGGCTTACGA][AGATCGGAAGAGC]
          ↑
          UMI

Extract and save:

UMI = GATTACGA

The UMI may be stored in the read name, read header or other metadata, depending on the pipeline.

For example:

Read ID: read001
UMI: GATTACGA

The sequence itself may later be removed, but the saved UMI can still be used during deduplication.

# 4. Which parts are artificial sequences?

The read can be divided into artificial and biological components:
[barcode][UMI][fixed sequence][biological insert][adapter]

In the example:
[ACGTAC][GATTACGA][TTT][CCGGATGCTAACGTAGGCTTACGA][AGATCGGAAGAGC]

The functions of each part are:
Sequence component	         Main function	                                           Used for mapping?
Barcode	Read                 identification or terminal-read assignment	               No
Fixed sequence	             Read assignment and structure validation	               No
UMI	                         Identification of original molecules and deduplication	   Saved separately
Adapter/primer	             Enables library construction or sequencing	               No
Biological insert	         Sequence originating from the biological sample	       Yes


Barcode and fixed sequence
→ used for 3P/5P assignment and then removed

UMI
→ extracted, saved and then removed from the read sequence

Adapter and primer
→ removed before mapping

Biological insert
→ retained for STAR mapping


# 5. Remove the barcode, UMI and fixed sequence
For the example 3P read, the artificial structure at the 5′ end is:   6 bp barcode + 8 bp UMI + 3 bp fixed sequence = 17 bp
The first 17 bp are: ACGTACGATTACGATTT

A simplified fixed-position trimming command is:
cutadapt -u 17 \
    -o sample_3P_position_trimmed_R1.fastq.gz \
    sample_3P_R1.fastq.gz

The option: -u 17
means:
Remove 17 bases from the 5′ end of each read.
Before trimming:
[ACGTAC][GATTACGA][TTT][CCGGATGCTAACGTAGGCTTACGA][AGATCGGAAGAGC]
After removing the first 17 bp:
[CCGGATGCTAACGTAGGCTTACGA][AGATCGGAAGAGC]
The remaining sequence is:
CCGGATGCTAACGTAGGCTTACGAAGATCGGAAGAGC

At this stage:
Barcode        → removed
UMI            → already extracted and saved
TTT sequence   → removed
Biological insert → retained
Adapter        → still present

5.1 3P and 5P reads should not be trimmed identically
In the simplified structures:
3P artificial 5′ structure
= 6 bp barcode + 8 bp UMI + 3 bp fixed sequence = 17 bp

5P artificial 5′ structure = 4 bp fixed sequence + 8 bp UMI + 8 bp fixed sequence = 20 bp

Therefore, the simplified commands would be different.
For 3P reads:
cutadapt -u 17 \
    -o sample_3P_trimmed_R1.fastq.gz \
    sample_3P_R1.fastq.gz

For 5P reads:
cutadapt -u 20 \
    -o sample_5P_trimmed_R1.fastq.gz \
    sample_5P_R1.fastq.gz

Internal reads do not necessarily contain the same terminal structure.

Therefore: 3P, 5P and internal reads should not be blindly trimmed using the same fixed number of bases.

The trimming rule must follow the actual read structure and the pipeline profile.

Do not manually remove these regions again unless it has been confirmed that the pipeline has not already removed them.

Repeated trimming can:
Remove part of the true biological insert
Make the reads unnecessarily short
Reduce the STAR mapping rate
Increase the proportion of reads reported as unmapped: too short

# 6. Remove the adapter sequence
Adapters are different from barcode and fixed-position sequences.
The barcode, UMI and terminal fixed sequence appear at predefined positions. In contrast, an adapter may appear at different positions depending on the length of the biological insert.
A simplified sequencing fragment is:
[adapter][biological insert][adapter]

When the biological insert is shorter than the sequencing read length, the sequencer can read through the insert and enter the adapter:
[short biological insert][adapter]

Therefore, adapter trimming is usually sequence-based rather than position-based.
In the example, after removing the first 17 bp, the sequence is:
[CCGGATGCTAACGTAGGCTTACGA][AGATCGGAAGAGC]

Cutadapt searches for the known adapter:
AGATCGGAAGAGC

A simplified command is:
cutadapt \
    -a AGATCGGAAGAGC \
    -m 20 \
    -o sample_3P_trimmed_R1.fastq.gz \
    sample_3P_position_trimmed_R1.fastq.gz

Before adapter trimming:
[CCGGATGCTAACGTAGGCTTACGA][AGATCGGAAGAGC]

After adapter trimming:
CCGGATGCTAACGTAGGCTTACGA

The remaining 24 bp sequence is the biological insert:
CCGGATGCTAACGTAGGCTTACGA

6.1 Fixed-position trimming versus sequence-based trimming
Fixed-position trimming

Example:
cutadapt -u 17

This means:
Always remove the first 17 bp.
It is suitable when an artificial structure is known to occupy fixed positions.

Sequence-based trimming
Example:
cutadapt -a AGATCGGAAGAGC

This means:
Search for the specified adapter sequence and remove it when detected.
It is suitable when the artificial sequence can appear at different positions.

The main difference is:
-u
→ remove a fixed number of bases

-a, -A, -g or -G
→ search for a known sequence and trim according to where it is found

# 7. Paired-end adapter or primer trimming

For paired-end reads, R1 and R2 can be processed together.

Example of 3′ adapter trimming:

cutadapt \
    -a ADAPTER_SEQUENCE_R1 \
    -A ADAPTER_SEQUENCE_R2 \
    -m 20 \
    -o sample_trimmed_R1.fastq.gz \
    -p sample_trimmed_R2.fastq.gz \
    sample_R1.fastq.gz \
    sample_R2.fastq.gz

The options mean:

-a
Search for and remove the 3′ adapter from R1.

-A
Search for and remove the 3′ adapter from R2.

-m 20
Discard reads or read pairs that are shorter than 20 bp after trimming.

-o
Write the trimmed R1 reads to this output file.

-p
Write the trimmed R2 reads to this output file.

If a primer or artificial sequence is expected at the 5′ end, -g and -G may be used:

cutadapt \
    -g PRIMER_SEQUENCE_R1 \
    -G PRIMER_SEQUENCE_R2 \
    -m 20 \
    -o sample_trimmed_R1.fastq.gz \
    -p sample_trimmed_R2.fastq.gz \
    sample_R1.fastq.gz \
    sample_R2.fastq.gz

Here:

-g
Search for a 5′ primer or adapter in R1.

-G
Search for a 5′ primer or adapter in R2.

This is different from using -u, because Cutadapt searches for a sequence instead of automatically deleting a fixed number of bases.

# 8. Minimum-length filtering

The example biological insert after trimming is:

CCGGATGCTAACGTAGGCTTACGA

Its length is:

24 bp

Because the command contains:

-m 20

the read is retained:

24 bp ≥ 20 bp
→ keep the read

Suppose another raw read becomes the following after barcode, UMI, fixed-sequence and adapter removal:

TCCGGG

Its length is only:

6 bp

Therefore:

6 bp < 20 bp
→ discard the read

Such a short sequence usually cannot be mapped reliably to a unique genomic location.

The purpose of -m 20 is not to require that the original FASTQ read be longer than 20 bp.

It means:

The read must still contain at least 20 bp after all trimming steps.
# 9. Other important Cutadapt parameters

The profile may contain options such as:

--rc -m 20 -j 0 -n 4
-m 20
Keep only reads that are at least 20 bp long after trimming.

Reads shorter than 20 bp are filtered as too short.

--rc
Also check the reverse-complement orientation.

This can be useful when the orientation of a sequence is uncertain.

However, it can also make trimming more aggressive, so the reason for using it should be understood from the library design.

-n 4
Perform up to four rounds of adapter trimming.

This may help remove several artificial sequences from the same read.

However, overly aggressive repeated trimming may remove too much sequence.

-j 0
Automatically use the available CPU cores.

This mainly affects processing speed rather than the biological trimming rule.

# 10. Complete sequence transformation

The complete example can be summarized as:

Raw read

[ACGTAC][GATTACGA][TTT][CCGGATGCTAACGTAGGCTTACGA][AGATCGGAAGAGC]
    │         │       │                │                    │
 barcode     UMI    fixed seq     biological insert       adapter

        │
        ├── Match barcode and TTT at expected positions
        │
        └── Assign as a 3P read
        ↓

Extract UMI

UMI = GATTACGA

        │
        ├── Save the UMI in metadata
        │
        └── Remove the first 17 bp
        ↓

After fixed-position trimming

[CCGGATGCTAACGTAGGCTTACGA][AGATCGGAAGAGC]

        │
        └── Search for and remove adapter
        ↓

After adapter trimming

CCGGATGCTAACGTAGGCTTACGA

        │
        ├── Length = 24 bp
        └── Passes -m 20
        ↓

Sequence entering STAR

CCGGATGCTAACGTAGGCTTACGA

The purpose of trimming is not simply to make reads shorter.

The purpose is to:

Remove artificial sequences
Preserve the reliable biological insert
Prevent artificial sequences from interfering with mapping
Retain enough biological sequence for reliable alignment
# 11. How does the mapping work?

After read assignment, UMI extraction and trimming, the remaining biological sequence is sent to STAR for genome alignment.

A good mapping result means:

```text
Clean biological reads
→ align to the expected genomic regions
→ with enough matched bases
→ without relying on excessively loose parameters
```

A poor mapping result may occur when:

```text
Reads still contain adapter / primer / polyA-polyT / other artificial sequence

The true biological insert is too short

The read is low-complexity

The read orientation is incorrect

The reference genome is incomplete or incorrect

The mapping parameters are not suitable for the library
```

The goal is therefore not simply to obtain the highest possible mapping rate.

The goal is to obtain:

```text
Biologically meaningful
+
reliable
+
sufficiently specific
alignments
```

---

## 11.1 Continue the previous 3P example

From the previous trimming example, the raw read was:

```text
[ACGTAC][GATTACGA][TTT][CCGGATGCTAACGTAGGCTTACGA][AGATCGGAAGAGC]
```

After:

```text
3P assignment
→ UMI extraction
→ removal of barcode + UMI + fixed sequence
→ adapter trimming
```

the remaining biological sequence is:

```text
CCGGATGCTAACGTAGGCTTACGA
```

This is the sequence that should enter STAR:

```text
STAR input:

CCGGATGCTAACGTAGGCTTACGA
```

STAR should not map:

```text
ACGTAC
GATTACGA
TTT
AGATCGGAAGAGC
```

because these are barcode, UMI, fixed sequence or adapter rather than biological genomic sequence.

---

# 12. How does STAR find the genomic position of a read?

The mapping process can be simplified into seven steps.

## Step 1. Take a cleaned read

For example:

```text
CCGGATGCTAACGTAGGCTTACGA
```

This read has already passed:

```text
read assignment
UMI extraction
trimming
minimum-length filtering
```

---

## Step 2. Split the read into shorter searchable pieces

STAR uses shorter parts of the read as searchable pieces, often referred to conceptually as seeds.

For example, the read:

```text
CCGGATGCTAACGTAGGCTTACGA
```

can be conceptually thought of as containing shorter pieces such as:

```text
CCGGATGC
GCTAACGT
GTAGGCTT
GCTTACGA
```

These are only illustrative pieces used to understand the idea.

The exact internal seed-searching procedure is handled by STAR.

The important concept is:

```text
STAR does not blindly compare the entire read against every possible genomic position.

It first uses shorter searchable sequence information to identify candidate genomic regions.
```

---

## Step 3. Search the reference genome

STAR searches for genomic locations that contain sequence similar to the searchable parts of the read.

Conceptually:

```text
Read:

CCGGATGCTAACGTAGGCTTACGA

        ↓

Reference genome:

...ATCGCCGGATGCTAACGTAGGCTTACGATCCA...
       └────────── match ──────────┘
```

If the sequence is found at one strong genomic location, the read may become uniquely mapped.

If similar sequence occurs at several locations, it may become multi-mapped.

---

## Step 4. Build candidate alignments

After possible genomic regions are identified, STAR constructs candidate alignments.

Conceptually:

```text
Read:
CCGGATGCTAACGTAGGCTTACGA

Candidate genomic region:
CCGGATGCTAACGTAGGCTTACGA
```

This is an ideal full match.

However, real sequencing reads may contain:

```text
mismatches
insertions
deletions
splice junctions
soft-clipped bases
```

Whether these are accepted depends on the mapping parameters.

---

## Step 5. Allow gaps or splice junctions

RNA-seq reads may cross exon–exon junctions.

Therefore, STAR can align a single read to two genomic regions separated by an intron.

Conceptually:

```text
Read:

AAAAACCCCCGGGGGTTTTT
```

may correspond to:

```text
Exon 1             Exon 2
AAAAACCCCC         GGGGGTTTTT
      \             /
       \---intron--/
```

The read contains the joined RNA sequence, while the reference genome contains an intron between the two exons.

STAR therefore needs splice-junction-related parameters to determine which gaps are biologically plausible.

---

# 13. Score and filter candidate alignments

STAR may find several possible alignments for one read.

It therefore has to decide whether each alignment is sufficiently reliable.

The result depends on factors such as:

```text
How much of the read can be aligned

How many bases match

How many mismatches are present

Whether gaps or splice junctions are acceptable

Whether the same read aligns to multiple genomic locations
```

After filtering, each read is generally reported as:

```text
Uniquely mapped

Multi-mapped

Unmapped
```

---

# 14. Main STAR mapping output

A key STAR output file is:

```text
*_star_Log.final.out
```

This file summarizes the final mapping statistics.

The most important values to inspect include:

```text
Number of input reads

Uniquely mapped reads %

% of reads mapped to multiple loci

% of reads unmapped: too short
```

---

# 15. How to interpret the mapping statistics

## 15.1 Number of input reads

This tells us how many reads actually entered STAR.

It is important because the number entering STAR may already be much lower than the original FASTQ read count due to:

```text
read classification
trimming
minimum-length filtering
other preprocessing
```

Therefore, when interpreting a low number of mapped reads, first ask:

```text
Did STAR receive enough reads in the first place?
```

---

## 15.2 Uniquely mapped reads %

This represents reads that map to one reliable genomic location.

Conceptually:

```text
Read
   ↓
Only one strong genomic location
   ↓
Uniquely mapped
```

This is usually one of the most important mapping quality metrics.

A higher uniquely mapped percentage generally means that more reads contain enough usable biological sequence to be assigned to a specific genomic position.

However, it should always be interpreted together with the library design and the other mapping statistics.

---

## 15.3 Reads mapped to multiple loci

These reads align to more than one genomic location.

Conceptually:

```text
Read
   ↓
Genome position A  ← good match
Genome position B  ← good match
Genome position C  ← good match
   ↓
Multi-mapped
```

This can happen when reads are:

```text
short

low-complexity

derived from repetitive regions

derived from repeated genomic sequences
```

For example, a very short sequence such as:

```text
ATATATATATATAT
```

may occur at many genomic positions.

In this case, STAR cannot confidently assign it to only one location.

---

## 15.4 Unmapped: too short

This metric requires careful interpretation.

```text
Unmapped: too short
```

does **not** simply mean:

```text
The original FASTQ read was physically short.
```

Instead, it means that STAR could not find a sufficiently long and reliable alignment for the read.

For example, the original read may be:

```text
100 bp
```

but if most of the sequence consists of:

```text
adapter
primer
polyA/polyT
barcode
other artificial sequence
```

then only a small part may be useful for genomic alignment.

Conceptually:

```text
Original read
100 bp

        ↓ trimming / alignment

Only 12 bp can be reliably matched to the genome

        ↓

STAR may report:

unmapped: too short
```

Possible reasons include:

```text
The true insert is too short

Adapter remains

Primer remains

polyA/polyT remains

The sequence is low-complexity

The read orientation is incorrect

The reference genome is incomplete or incorrect

Mapping parameters are not suitable for this library
```

---

# 16. Example from the MicEdges mapping results

The notes record the following mapping percentages:

```text
3P uniquely mapped      = 0.04%

5P uniquely mapped      = 3.66%

Internal uniquely mapped = 7.36%
```

At the same time:

```text
3P unmapped: too short      = 99.91%

5P unmapped: too short      = 95.09%

Internal unmapped: too short = 89.95%
```

The main pattern is therefore:

```text
3P
very low uniquely mapped
+
almost all reads unmapped as too short

5P
slightly better than 3P
but still dominated by unmapped: too short

Internal
best among the three groups
but still has a very high unmapped: too short percentage
```

The interpretation recorded in the notes is:

```text
The cleaned reads, especially the 3P reads,
do not contain enough usable genomic sequence
for reliable STAR mapping.
```

---
# 17. Connecting trimming results with STAR results

The trimming and mapping steps should not be interpreted independently.

They form one continuous process:

```text
Raw read
   ↓
Read assignment
   ↓
UMI extraction
   ↓
Trim artificial sequence
   ↓
How much biological sequence remains?
   ↓
STAR mapping
   ↓
Uniquely mapped / multi-mapped / unmapped
```

For example:

### Case 1: good trimming

```text
Raw read:

[barcode][UMI][fixed seq][40 bp biological insert][adapter]

        ↓ trimming

40 bp biological insert

        ↓ STAR

Enough usable sequence
→ potentially reliable mapping
```

### Case 2: true insert is very short

```text
Raw read:

[barcode][UMI][fixed seq][8 bp insert][adapter]

        ↓ trimming

8 bp biological insert

        ↓ STAR

Too little usable sequence
→ likely unmapped
```

### Case 3: adapter is not completely removed

```text
After trimming:

[15 bp biological insert][adapter sequence remains]

        ↓ STAR

Only part of the read corresponds to the genome

        ↓

Poor alignment
or
unmapped: too short
```

Therefore, a high:

```text
unmapped: too short
```

value should make us look back at the earlier preprocessing steps.

---

# 18. How should STAR parameters be designed for our home-made data?

The notes divide STAR parameter design into four main categories:

```text
1. Parameters related to the reference genome

2. Parameters related to read length and matched bases

3. Parameters related to splice junctions and introns

4. Parameters related to soft clipping
```

The important principle is:

```text
STAR parameters should reflect the real structure and quality of the home-made library.
```

They should not simply be made increasingly permissive until the mapping rate becomes high.

A high mapping rate obtained using overly loose parameters may include unreliable alignments.

Therefore:

```text
Good mapping
≠ simply maximizing mapped reads

Good mapping
= enough matched biological sequence
+ reasonable alignment quality
+ biologically plausible genomic positions
```

---

## 18.1 Reference-genome-related parameters

Before changing read-alignment thresholds, first confirm that the correct reference is being used.

The profile should be checked for settings related to:

```text
ReferenceGenome

GenomeFasta

GeneAnnotationGTF
```

A wrong or incomplete reference can cause otherwise valid reads to fail mapping.

A useful profile check is:

```bash
grep -nE "^(ReferenceGenome|GenomeFasta|GeneAnnotationGTF)" \
    profile_name.profile
```

The purpose of this command is to quickly locate the reference-related settings in the profile.

---

## 18.2 Read-length and matched-base-related parameters

For short biological inserts, the amount of sequence that STAR can align becomes especially important.

The key question is:

```text
After trimming, how many real biological bases remain?
```

For example:

```text
Raw read = 100 bp

Artificial sequence removed = 80 bp

Remaining biological insert = 20 bp
```

This read has a very different mapping situation from:

```text
Raw read = 100 bp

Artificial sequence removed = 20 bp

Remaining biological insert = 80 bp
```

Therefore, STAR parameters should be interpreted together with the post-trimming read length.

---

## 18.3 Splice-junction and intron parameters

Because this is RNA-seq data, some reads may span splice junctions.

Therefore, STAR must determine whether a gap in the genomic alignment represents a plausible intron.

Conceptually:

```text
RNA read:

Exon 1 + Exon 2

Genome:

Exon 1 ---- intron ---- Exon 2
```

If splice-junction parameters are inappropriate, valid RNA reads may fail to align correctly.

---

## 18.4 Soft clipping

Sometimes only part of a read can be aligned reliably.

Conceptually:

```text
Read:

XXXXXXCCGGATGCTAACGTAGGCTTACGA
      └──── genomic match ─────┘
```

The unmatched terminal bases may be treated differently depending on alignment settings.

Soft clipping can allow part of the read to align while leaving terminal bases unaligned.

However, very permissive soft clipping can also make poor-quality reads appear to map.

Therefore, soft clipping should not be used simply to rescue every read.

---

# 19. How to remove duplication?

After STAR mapping, the next important step for the terminal reads is **deduplication**.

The complete logic is:

```text
Original RNA/cDNA molecule
        ↓
Add UMI
        ↓
PCR amplification
        ↓
One original molecule may generate many sequencing reads
        ↓
STAR mapping
        ↓
Compare mapping position + UMI
        ↓
Collapse PCR copies
        ↓
Keep one representative molecule
```

The main purpose is to prevent one original molecule from being counted many times simply because it was amplified efficiently during PCR.

UMIs are particularly useful because two biological molecules can genuinely produce reads at the same genomic position. A UMI provides additional information that helps distinguish:

```text
same genomic position + same UMI
→ probably copies of the same original molecule

same genomic position + different UMI
→ probably different original molecules
```

Removing duplicates based only on genomic coordinates can therefore remove genuine biological molecules from RNA-seq data; UMI-based deduplication provides a way to distinguish PCR duplicates from biologically independent molecules.

---

# 20. Why is deduplication performed after mapping?

This order is very important.

The workflow is:

```text
Raw FASTQ
   ↓
Extract UMI
   ↓
Save UMI in read name
   ↓
Trim artificial sequences
   ↓
STAR mapping
   ↓
Know genomic position
   ↓
UMI deduplication
```

UMI extraction must happen **before mapping**, because the UMI is part of the artificial sequence and should not be aligned to the genome.

However, deduplication happens **after mapping**, because UMI-tools needs both:

```text
UMI information
+
mapping coordinates
```

to decide whether two reads represent the same original molecule.

UMI-tools explicitly defines `dedup` as deduplication based on mapping coordinates and the UMI attached to the read.

So:

```text
UMI extraction
≠
UMI deduplication
```

They are two different steps.

### UMI extraction

```text
Where is the UMI?
↓
Take it out of the biological sequence
↓
Save it in metadata/read name
```

### UMI deduplication

```text
Where did the read map?
+
What UMI does it have?
↓
Determine whether it is a PCR copy
```

---

# 21. Continue the previous concrete example

Suppose the original biological molecule was:

```text
CCGGATGCTAACGTAGGCTTACGA
```

and before PCR it received:

```text
UMI = GATTACGA
```

Therefore:

```text
Original molecule A

UMI:
GATTACGA

Biological sequence:
CCGGATGCTAACGTAGGCTTACGA
```

During PCR, this one molecule may be amplified many times:

```text
Original molecule A
UMI = GATTACGA
        │
        ├── PCR copy 1
        ├── PCR copy 2
        └── PCR copy 3
```

After sequencing and mapping:

```text
Read 1
position = chrI:1000
UMI = GATTACGA

Read 2
position = chrI:1000
UMI = GATTACGA

Read 3
position = chrI:1000
UMI = GATTACGA
```

These reads have:

```text
same mapping position
+
same UMI
```

so they are interpreted as PCR copies of one original molecule.

Deduplication changes:

```text
3 sequencing reads
↓
1 representative molecule
```

Conceptually:

```text
Before dedup:

chrI:1000 + GATTACGA
chrI:1000 + GATTACGA
chrI:1000 + GATTACGA

        ↓ UMI dedup

After dedup:

chrI:1000 + GATTACGA
```

---

# 22. What if two reads map to the same position but have different UMIs?

Suppose another independent RNA molecule happens to produce the same sequence.

It receives another UMI:

```text
Molecule A:
chrI:1000
UMI = GATTACGA

Molecule B:
chrI:1000
UMI = CCGTAATC
```

After PCR:

```text
chrI:1000 + GATTACGA
chrI:1000 + GATTACGA
chrI:1000 + GATTACGA

chrI:1000 + CCGTAATC
chrI:1000 + CCGTAATC
```

After deduplication:

```text
chrI:1000 + GATTACGA
chrI:1000 + CCGTAATC
```

So:

```text
5 sequencing reads
↓
2 original molecules
```

This is the main reason that UMI-based deduplication is better suited to this situation than simply saying:

```text
same genomic position
= duplicate
```

because two genuinely different molecules may map to exactly the same location.

---

# 23. What does RNAEdgeFlow actually do?

This is important because the actual pipeline is more specific than the general principle.

RNAEdgeFlow currently performs:

```text
3P
→ UMI extraction
→ paired-end STAR mapping
→ UMI deduplication

5P
→ UMI extraction
→ paired-end STAR mapping
→ UMI deduplication

Internal
→ no terminal UMI extraction
→ paired-end STAR mapping
→ NO UMI deduplication
```

This is directly implemented in the current RNAEdgeFlow pipeline. The README also describes UMI extraction and deduplication as part of RNAEdgeFlow.

The actual 3P UMI extraction command is:

```bash
umi_tools extract \
    --bc-pattern="${P3_BC_Pattern}" \
    -I "${f_3p_r1}" \
    --read2-in "${f_3p_r2}" \
    --stdout "${f_3p_umi_r1}" \
    --read2-out "${f_3p_umi_r2}"
```

For 5P:

```bash
umi_tools extract \
    --bc-pattern="${P5_BC_Pattern}" \
    -I "${f_5p_r1}" \
    --read2-in "${f_5p_r2}" \
    --stdout "${f_5p_umi_r1}" \
    --read2-out "${f_5p_umi_r2}"
```

The current pipeline does **not** perform an equivalent UMI extraction step for the internal reads.

---

# 24. How does UMI extraction work in paired-end reads?

Our data are paired-end:

```text
fragment
  ├── R1
  └── R2
```

The UMI structure is detected from the terminal read structure in R1.

For example:

```text
R1:

[barcode][UMI][fixed sequence][biological insert...]

R2:

[sequence from the other side of the same fragment...]
```

UMI-tools removes the UMI from the sequence but retains the UMI information in the read name.

Its paired-end workflow keeps R1 and R2 associated with the same molecule. The UMI-tools documentation specifically recommends supplying both R1 and R2 during extraction for paired-end sequencing.

Conceptually:

```text
Before extraction

R1:
read001
[barcode][GATTACGA][fixed][insert...]

R2:
read001
[insert...]

        ↓ umi_tools extract

After extraction

R1:
read001_GATTACGA
[cleaner sequence...]

R2:
read001_GATTACGA
[sequence...]
```

The exact read-name appearance depends on UMI-tools formatting, but the important concept is:

```text
UMI no longer needs to remain in the sequence

but

UMI information remains associated with the read pair
```

UMI-tools normally expects the extracted UMI at the end of the read name when deduplication is later performed.

---

# 25. How does RNAEdgeFlow perform UMI deduplication?

The actual RNAEdgeFlow command is:

```bash
umi_tools dedup \
    -I "${inbam}" \
    -S "${dedup}" \
    --method=unique \
    --paired
```

and it is run only for:

```text
3P
5P
```

The options mean:

```text
-I
input BAM file

-S
output deduplicated BAM file

--method=unique
use exact UMI identity when defining UMI groups

--paired
the BAM contains paired-end reads
and the pair/template information should be considered
```

---

# 26. What does `--method=unique` mean?

This parameter is very important.

RNAEdgeFlow explicitly uses:

```bash
--method=unique
```

UMI-tools first groups reads according to mapping position. With the `unique` method, reads within the relevant mapping group must share the **exact same UMI** to be grouped together.

For example:

```text
Position: chrI:1000

Read A:
UMI = GATTACGA

Read B:
UMI = GATTACGA

Read C:
UMI = GATTACGA

Read D:
UMI = CCGTAATC
```

With:

```text
--method=unique
```

the result is conceptually:

```text
GATTACGA group
A + B + C
→ one molecule

CCGTAATC group
D
→ one molecule
```

Final molecule count:

```text
2
```

---

---
# 27. SE versus PE deduplication

## 27.1 Single-end reads

For a single-end read, the simplified information is:

```text
Read
↓
mapping position
+
orientation
+
UMI
```

Conceptually:

```text
Read A:
chrI:1000, + strand, UMI=GATTACGA

Read B:
chrI:1000, + strand, UMI=GATTACGA

→ duplicates
```

A simplified UMI-tools command for SE data would be:

```bash
umi_tools dedup \
    -I mapped.bam \
    -S deduplicated.bam \
    --method=unique
```

There is no:

```text
--paired
```

because there is only one sequencing read per fragment.

This is a general UMI-tools example, **not the current MicEdges RNAEdgeFlow command**.

---

## 27.2 Paired-end reads

For paired-end sequencing:

```text
Original fragment
   ├── R1
   └── R2
```

the pair gives more information about the original molecule.

Conceptually:

```text
R1 mapping position
+
R2 mapping position
+
fragment/template structure
+
UMI
```

can be used to define duplicate fragments more specifically.

UMI-tools states that `--paired` outputs both members of the pair and uses template length when determining reads with the same mapping coordinates.

Therefore RNAEdgeFlow uses:

```bash
umi_tools dedup \
    -I mapped.bam \
    -S deduplicated.bam \
    --method=unique \
    --paired
```

---

# 28. A concrete PE example

Suppose two original molecules exist.

### Molecule A

```text
UMI = AAAAAAAA

R1 → chrI:1000
R2 → chrI:1150
```

PCR produces three copies:

```text
Pair A1:
chrI:1000 ↔ chrI:1150
UMI AAAAAAAA

Pair A2:
chrI:1000 ↔ chrI:1150
UMI AAAAAAAA

Pair A3:
chrI:1000 ↔ chrI:1150
UMI AAAAAAAA
```

These represent one original fragment:

```text
3 read pairs
↓ dedup
1 molecule
```

Now suppose Molecule B happens to have the same fragment coordinates:

```text
R1 → chrI:1000
R2 → chrI:1150
```

but:

```text
UMI = CCCCCCCC
```

Then after deduplication:

```text
chrI:1000 ↔ chrI:1150 + AAAAAAAA
→ keep one

chrI:1000 ↔ chrI:1150 + CCCCCCCC
→ keep one
```

Therefore:

```text
same mapping coordinates
+
different UMI

→ two original molecules
```

---

# 29. How does it work with 3′ and 5′ terminal reads?

For RNAEdgeFlow, 3P and 5P are treated similarly at the deduplication stage.

Earlier stages differ because their barcode/fixed-sequence structures are different:

```text
3P
different terminal structure
different UMI position

5P
different terminal structure
different UMI position
```

But after:

```text
assignment
→ UMI extraction
→ trimming
→ STAR mapping
```

both become mapped paired-end BAM files containing UMI information.

Then RNAEdgeFlow performs the same dedup command for each:

```bash
for tag in 3P 5P; do

    umi_tools dedup \
        -I "${inbam}" \
        -S "${dedup}" \
        --method=unique \
        --paired

done
```

So the logic is:

```text
3P:

terminal structure
→ extract 3P UMI
→ map
→ UMI dedup
→ 3P_final.bam


5P:

terminal structure
→ extract 5P UMI
→ map
→ UMI dedup
→ 5P_final.bam
```

---

# 30. Why are terminal reads especially suitable for UMI deduplication?

For terminal reads, many genuine biological molecules may naturally share the same transcript-end position.

For example, multiple RNA molecules can terminate at the same PAS or start from the same TSS.

Therefore:

```text
same 5′ genomic coordinate
```

or:

```text
same 3′ genomic coordinate
```

does **not automatically mean PCR duplicate**.

The UMI provides another dimension:

```text
same terminal coordinate + different UMI
→ potentially independent molecules

same terminal coordinate + same UMI
→ likely PCR copies of the same molecule
```

This is particularly important when the biological signal itself is concentrated at transcript ends.

---

# 31. How does it work with internal reads?

This part is different.

RNAEdgeFlow identifies internal reads as reads that were not assigned to the terminal 3P/5P groups.

The current code then:

```text
internal reads
↓
adapter trimming
↓
fastp
↓
STAR mapping
↓
internal_final.bam
```

but **does not run `umi_tools dedup` on the internal BAM**.

The relevant code is conceptually:

```text
3P STAR BAM
↓
UMI dedup
↓
3P_final.bam

5P STAR BAM
↓
UMI dedup
↓
5P_final.bam

internal STAR BAM
↓
rename
↓
internal_final.bam
```

The actual pipeline moves the internal STAR BAM directly to:

```text
sample_internal_final.bam
```

without a UMI-tools dedup step.

---

# 32. Why aren't internal reads UMI-deduplicated in the current pipeline?

The most direct explanation from the code is:

```text
3P/5P
→ have explicit UMI extraction

internal
→ no equivalent UMI extraction
```

Therefore, the pipeline has the UMI information required for terminal UMI deduplication, but it does not prepare internal reads in the same way.

This does **not** mean:

```text
internal reads never contain duplication
```

It means:

```text
current RNAEdgeFlow
does not perform the same UMI-based deduplication
for the internal group
```

This distinction is important.

---

# 33. Should we simply remove internal duplicates according to mapping position?

Not automatically.

A conventional duplicate-marking approach such as `samtools markdup` can compare mapped coordinates, orientation and, for paired reads, mate information. It can also optionally incorporate barcode information.

However:

```text
coordinate duplicate
≠ always PCR duplicate
```

especially in RNA-seq.

Independent RNA molecules can produce identical genomic coordinates.

Research using UMI-labelled RNA-seq has shown that deduplication based solely on mapping coordinates can remove biologically meaningful reads and introduce bias.

Therefore:

```text
Do NOT automatically run:

samtools markdup -r ...

on internal RNA-seq reads

just because duplicate reads exist.
```

First understand:

```text
Does the internal library contain usable UMIs?

What exactly should be considered one original molecule?

What is the downstream purpose?

Does the established pipeline intentionally retain these reads?
```

For the current RNAEdgeFlow workflow, the safest interpretation is:

```text
follow the current pipeline:

UMI dedup → 3P and 5P

no UMI dedup → internal
```

unless the library design or supervisor specifies otherwise.

---

# 34. UMI collision

UMIs are random sequences, but their number is finite.

For an 8 bp UMI:

```text
4 possible nucleotides per position

Number of possible UMIs:

4^8 = 65,536
```

Examples:

```text
AAAAAAAA
GATTACGA
CCGTAATC
TGCATGCA
...
```

Usually, different original molecules are expected to receive different UMIs.

However, by chance, two independent molecules can receive the same UMI.

This is called:

```text
UMI collision
```

Example:

```text
Molecule A:
position = chrI:1000
UMI = GATTACGA

Molecule B:
position = chrI:1000
UMI = GATTACGA
```

The deduplication algorithm may interpret them as:

```text
one molecule
```

although biologically they were two molecules.

Therefore:

```text
UMI
is not an absolutely unique molecular serial number
```

It is a random molecular label whose usefulness depends on:

```text
UMI length
library complexity
sequencing depth
number of molecules sharing a mapping position
```

---

# 35. UMI sequencing errors

Another problem is the opposite of UMI collision.

One original molecule may acquire apparent different UMIs because the UMI itself contains a sequencing error.

Example:

```text
True original molecule:

UMI = GATTACGA
```

Most PCR copies are read correctly:

```text
GATTACGA
GATTACGA
GATTACGA
GATTACGA
```

but one sequencing read contains:

```text
GATTACGG
```

Now the data contain:

```text
GATTACGA  × 4

GATTACGG  × 1
```

With the current RNAEdgeFlow setting:

```bash
--method=unique
```

the two exact UMI sequences are treated separately.

By contrast, UMI-tools' network-based methods can consider edit distance between UMIs; for example, the `directional` method considers relationships between similar UMIs and their abundances.

This gives an important QC question:

```text
Are nearby UMI sequences real independent molecules,
or are some caused by UMI sequencing errors?
```

---

# 36. `unique` versus `directional`

A simplified comparison:

```text
UMIs at the same mapping position:

AAAAAAAA   count = 20
AAAAAAAT   count = 1
CCCCCCCC   count = 5
```

### `unique`

```text
AAAAAAAA
AAAAAAAT
CCCCCCCC

→ three UMI groups
```

because all three strings are different.

### `directional`

The algorithm can construct a network between similar UMIs and use both edit distance and abundance relationships. Therefore the low-count:

```text
AAAAAAAT
```

may potentially be grouped with:

```text
AAAAAAAA
```

depending on the method's rules.

Current RNAEdgeFlow:

```text
--method=unique
```

Therefore, when documenting the pipeline, write:

```text
RNAEdgeFlow currently performs exact-UMI deduplication,
rather than error-aware network-based UMI correction.
```

---

# 37. Which read is retained from a duplicate group?

Suppose UMI-tools identifies:

```text
Read A
Read B
Read C
```

as belonging to the same duplicate group.

Only one representative read is retained.

UMI-tools uses alignment-related criteria such as mapping information and mapping quality when selecting the representative; if reads remain equivalent, one may be selected at random.

Conceptually:

```text
PCR duplicate group

Read A
Read B
Read C

        ↓

keep one representative read
```

The purpose is not to create a consensus sequence.

It is primarily:

```text
collapse one amplified molecule
into one representative observation
```

---

# 38. Deduplication does not mean deleting all repeated sequences

This distinction is very important.

Suppose:

```text
Read A:
sequence = ATGCC...
position = chrI:1000
UMI = AAAAAAAA

Read B:
sequence = ATGCC...
position = chrI:1000
UMI = CCCCCCCC
```

The biological sequence is identical.

But the UMIs differ.

Therefore:

```text
same sequence
≠ automatically duplicate
```

Conversely:

```text
Read A:
position = chrI:1000
UMI = AAAAAAAA

Read B:
position = chrI:1000
UMI = AAAAAAAA
```

These are much stronger candidates for being PCR copies.

Therefore deduplication is better thought of as:

```text
identify repeated observations
of the same original molecule
```

rather than:

```text
delete identical sequence strings
```

---

# 39. RNAEdgeFlow deduplication in the complete workflow

Now the complete pipeline can be extended from the previous section.

```text
RAW PE READS
      ↓
3P / 5P / internal assignment
      ↓
      ├──────────────────┬───────────────────┐
      ↓                  ↓                   ↓
     3P                 5P                internal
      ↓                  ↓                   ↓
extract UMI          extract UMI         no terminal
                                           UMI extraction
      ↓                  ↓                   ↓
trim                trim                trim
      ↓                  ↓                   ↓
fastp               fastp               fastp
      ↓                  ↓                   ↓
STAR PE mapping     STAR PE mapping     STAR PE mapping
      ↓                  ↓                   ↓
UMI dedup           UMI dedup           no UMI dedup
unique + paired     unique + paired
      ↓                  ↓                   ↓
3P_final.bam        5P_final.bam       internal_final.bam
```

This matches the current RNAEdgeFlow implementation.

---

# 40. What are the final BAM files?

After deduplication/finalisation, RNAEdgeFlow produces:

```text
sample_3P_final.bam

sample_5P_final.bam

sample_internal_final.bam
```

However, their meanings are slightly different.

### `3P_final.bam`

```text
3P reads
+
mapped
+
UMI-deduplicated
```

### `5P_final.bam`

```text
5P reads
+
mapped
+
UMI-deduplicated
```

### `internal_final.bam`

```text
internal reads
+
mapped
+
NOT UMI-deduplicated by the current pipeline
```

This difference should be remembered when comparing read numbers between the three groups.

---

# 41. BAM indexing

After generating the final BAM files, RNAEdgeFlow runs:

```bash
samtools index \
    --threads "${THREADS}" \
    sample_final.bam
```

This produces:

```text
sample_final.bam
sample_final.bam.bai
```

The `.bam` contains the mapped alignments.

The `.bai` is the BAM index.

It allows programs to quickly retrieve alignments from a specific genomic region without reading the entire BAM file.

The final RNAEdgeFlow code indexes all 3P, 5P and internal BAMs.

