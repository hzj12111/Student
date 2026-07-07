# 20260707

## How to assign the 5', 3' and internal reads? How to do it better and faster?
RNAEdgeFlow splits reads into 3P, 5P, internal, and uncertain groups using the P3/P5 barcode and fixed sequence settings. instead of mapping and then assigning.
If a read matches the 3P barcode/fixed sequence pattern at the expected position, it is classified as a 3P terminal read.

Raw read
Check if the first few bits of the read match the 3P pattern → If they match → assign as a 3P read
        If they don't match 3P → Check if it matches the 5P pattern → If it matches → assign as a 5P read
                  If it doesn't match the terminal pattern → Assign as an internal read
If it only partially matches the loose terminal rule → Assign as uncertain
If the reads don't match 3P/5P patterns, they will be assigned as internal reads

Example 3P settings:
3P read start:
6bp allowed barcode     8bp UMI            TTT
  positions 1–6          7–14             15–17
                   
P3_BC_Pattern=XXXXXXNNNNNNNNXXX
P3_UmiRegion=7:14                                          from 7 to 14 : 8bp UMI ,  1-based inclusive read positions
P3_FixedSeq1=
P3_FixedSeq1_File=./tests/p3_barcode_position1_6.txt
P3_FixedSeq1Region=1:6                                     Check bits 1–6 of the read command to see if they belong to the allowed sequences in p3_barcode_position1_6.txt.
P3_FixedSeq1Mismatch=0
P3_FixedSeq1MismatchLoose=
P3_FixedSeq2=TTT
P3_FixedSeq2Region=15:17
P3_FixedSeq2Mismatch=0
P3_FixedSeq2MismatchLoose=                                 FixedSeq*_Mismatch controls strict matching.
                                                           FixedSeq*_MismatchLoose controls loose matching for uncertain/diagnostic read classification.→ loose terminal → uncertain reads
                                                           uncertain = loose_terminal - strict_terminal 
                                                           some reads resemble terminal reads but do not meet the strict standard.

Example 5P settings:
5P read start:
AGTC      UMI, 8 bp       CATCAGGG
1–4        5–12           13–20

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





# 20260708
## How to cut the barcode and adaptor sequence? How to make it more sense? 

