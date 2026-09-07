# .huh???: A crash course in file formats
If you've spent any significant amount of time doing stuff on a computer, it's pretty likely you've dealt with files (in fact, you're reading a file right now, if you can believe it). I would even hazard a guess that you've had the occasional file format mishap or saw an unfamiliar file extension and went, "what is this" [sic]. The more computer-intensive your work, the more frequently you're probably going to have this experience. In bioinformatics, there are a few standardized file formats, and then there are a bunch of other ones, because the thing about file formats (especially for text files) is you can kind of just make up a format and call it whatever you want. That said, a file's format and extension can tell you a lot about the data within before you ever open it. Therefore, I think it's worth setting aside some time to talk about some common formats I dealt with during this project so we can better understand what they are and how they're meant to be used and interpreted. 

Most the files covered in this Markdown are, at their core, *plain text files*. (The exceptions are BAM, CRAM, BAI, CSI, and BCF, which are all binary alternatives to some of the text files discussed in this section. BGZF is also discussed.) In theory, you could open any of these files on any text editor and see their contents just fine. You probably wouldn't want to, though, because they're usually pretty large, and you're unlikely to get anything useful out of trying to read them that way. What sets the different formats apart from each other are the types of information they contain and how that information is organized. 

## FASTA
**Extensions**: `.fasta`, `.fa`, `.fas`, `.fna`, `.ffn`, `.faa`, `.mpfa`, `.frn`

When one begins working with nucleotide or amino acid sequences, the FASTA format is often the first format they will encounter. Named for the software package[^1] from which they originated, FASTA files are actually pretty simple in their structure. They consist of two main components:

1. the **header** or **description** line (also called the defline), which consists of a sequence identifier and optional descriptive information, and is preceded by a greater-than (`>`) symbol, and
2. the **sequence** itself, which can be nucleotides or amino acids, both of which are represented by the codes created and standardized by the IUPAC[^2].

And that's it, really. Here's an example, borrowed from a guide on NCBI's BLAST website[^3]:

```
>P01013 GENE X PROTEIN (OVALBUMIN-RELATED)
QIKDLLVSSSTDLDTTLVLVNAIYFKGMWKTAFNAEDTREMPFHVTKQESKPVQMMCMNNSFNVATLPAE
KMKILELPFASGDLSMLVLLPDEVSDLERIEKTINFEKLTEWTNPNTMEKRRVKVYLPQMKIEEKYNLTS
VLMALGMTDLFIPSANLTGISSAESLKISQAVHGAFMELSEDGIEMAGSTGVIEDIKHSPESEQFRADHP
FLFLIKHNPTNTIVYFGRYWSP
```

The header starts with the `>` symbol and is followed by the identifier, `P01013`. The rest of the line is a human-readable description of what the sequence actually is (in this case, the ovalbumin-related protein X). By the way, the format of the sequence identifier (SeqID) varies depending on where the sequence came from. Wikipedia has a table summarizing the SeqID format standards defined by the NCBI here[^4].

The next lines contain the amino acid sequence. Sequences can be multiline or single-line, but blank lines in the middle of the sequence are illegal. In the context of this project, it should be noted that the GATK ignores non-standard bases[^5].

Sometimes, a FASTA file can contain multiple sequences. Every sequence will follow the same format outlined above, so it might look something like this:

```
>sequence_1
ATGCATGCATGCATGCATGCATGCATGCAT
>sequence_2
GCATGCATGCATGCATGCATGCATGCATGC
...
```

Now, let's talk about the many extensions I listed at the top of this section. As near as I can tell, `.fasta` and `.fa` are more or less the most commonly used extensions for generic FASTA files (though `.fas` also exists, I guess), and the rest are used to be a bit more specific about the nature of the sequence within the file. For example, `.fna` specifically contains nucleic acids (**F**ASTA **N**ucleic **A**cid), while `.faa` contains amino acids (**F**ASTA **A**mino **A**cid). Wikipedia has another table describing the various extensions[^6]. 

Because FASTA files are often very large, the GATK also requires the FASTA to be accompanied by an index file and a dictionary file, "because it allows efficient random access to the reference bases"[^5]. The dictionary format I'll get to later, because its structure is predicated on a SAM-style header, so I think it behooves us to understand SAM files before we talk about DICT files. The index format is described below. 

### FASTA/FASTQ Index (FAI)
**Extension**: `.fai` (appended to the end of a FASTA or FASTQ filename, including the original extension)

Let's pretend your FASTA file is a book, and you want to find the page of a certain sequence. The naive way to go about this would be to start from the very beginning and flip through each page individually until you find what you are looking for. If you're lucky, you might find it within the first few pages. If you're not, well, it's a very long book, so you'll be sitting there looking for a while. Now imagine doing that every time you needed to look up anything. That would be terribly inefficient and a generally unpleasant way to live your life. But you don't have to live this way.

The same way you use an index in a book to quickly find certain sections containing the information you want, the purpose of a FASTA index is to facilitate efficient random lookups of sequences and subsequences in the associated FASTA file. The FAI maintains a record of things like byte offsets, (sub)sequence lengths, and the per-line numbers of bases/bytes, and each set of information is linked to a sequence by its identifier[^7]. Downstream tools can then extract specific regions of the FASTA without having to search through the entire thing every time, which is a highly significant advantage when you are working with large reference genomes. Take some extra time to generate the FAI file once, and the rest of your analysis will be that much faster. In fact, life is so much better with a FASTA index that GATK *requires* one to be present for most of its tools to run[^5].

The FASTA index file is a headerless, tab-delimited text file consisting of five fields per sequence[^7] [^8]:

1. `NAME`: the sequence identifier
2. `LENGTH`: the total number of bases in the sequence
3. `OFFSET`: the byte offset in the FASTA pointing to the beginning of the sequence (0-based, beginning at the start of the file and including the newline); in lay terms, this is equivalent to the page number in a book index
4. `LINEBASES`: the number of *bases per line* of sequence data
5. `LINEWIDTH`: the number of *bytes per line* of sequence data (including the newline)

Since the FAI file doesn't contain a header, the fields aren't labeled, so you just have to know what the numbers mean. This also means the field names aren't concrete; for example, the GATK team refers to these fields as the contig, size, location, basesPerLine, and bytesPerLine, respectively[^5]. 

Importantly, the FAI file can only be used in conjunction with the *exact version* of the reference FASTA it was generated from. If you make any changes to the FASTA at all, you need to generate a new FAI from the updated version to keep things consistent. 

I think the byte offset is a potential area of confusion, so let me explain it a bit further: Counting the offset starts at the *very beginning* of the file, meaning it also counts every character in the header line, including the newline character at the end. Here's a slightly modified example I borrowed from the Samtools `faidx` page[^8] to illustrate this:

Say the beginning of your FASTA file looks like this.

```
>chr1
ATGCATGCATGCATGCATGCATGCATGCAT
GCATGCATGCATGCATGCATGCATGCATGC
ATGCAT
```

Since the offset is 0-based, we count the `>` character as 0 and continue from there:

`>` -> 0\
`c` -> 1\
`h` -> 2\
`r` -> 3\
`1` -> 4\
`\n` -> 5

Meaning the first base in the `chr1` sequence in the FAI file will be recorded as 6. Now, keep in mind that in Unix-style (LF) line termination, the newline counts as **one** byte. If you were to use Windows-style (CR-LF) line termination, the newline would be **two** bytes[^7] [^8]. 

Since the sequence is 66 bases long but is split across three lines, the `LINEBASES` and `LINEWIDTH` values will be 30 and 31, respectively (since `LINEWIDTH` counts the newline at the end of each line). Even though the last line is only 6 bases long, it's not counted toward these fields because it's *incomplete*, and the reader only needs to know how long a *complete* line is so it knows how to skip them without messing anything up. It's kind of like knowing a certain page of a book is within a specific chapter. If you know what page number you're looking for and which chapter it's in, you don't need to count from the beginning. You can skip the preceding chapters until you land on the correct one, and then you flip through to the page you want. It's just an additional way of speeding up the lookup process. This does require that the FASTA file has consistent line lengths (minus the last line, which can be shorter) to work. 

The FAI entry should look like this: 

```
chr1    66    6    30    31
```

By the way, all of this applies pretty much exactly the same with FASTQ files, except there's a sixth field, `QUALOFFSET`, which is like `OFFSET` but for the quality line (and starts after the newline following the first `+`). Speaking of FASTQ files ...

## FASTQ
**Extension**: `.fastq`

Conceptually, a FASTQ file is the same as a FASTA file, except it includes per-base quality scores. It consists of exactly four line-separated fields (so no multiline sequences, unlike the FASTA format), which are the following[^9]: 

1. The **header line** with the sequence identifier, just like the FASTA defline (including the optional description), except it's preceded by an 'at' (`@`) symbol instead of a `>` symbol
2. The raw **sequence** letters 
3. A **separator line** consisting of a plus sign (`+`) and, optionally, the identifier again
4. The Phred quality scores for each base, encoded as ASCII characters; the length of this line must *exactly* match the length of the sequence line (*i.e*, there must be one quality score character for each nucleotide)

I'm not going to get into what Phred quality scores actually are, but here's the Wikipedia article[^10] if you're not familiar and/or just curious. The point is that they're the standard measure of base quality, which essentially refers to how accurately the sequencer called each base across the reads. In order to have exactly one character representing quality per base, the quality scores are encoded as printable ASCII characters. This is achieved by adding a baseline offset to the ASCII values. In Illumina 1.8 and later (as well as pretty much all modern sequencing platforms), that offset is **33**, and the quality scores range from **0 to 93**; this encoding is called Sanger or Phred+33 encoding[^9] [^11]. Therefore, the range of ASCII characters used for base quality scores are:

```
!"#$%&'()*+,-./0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwxyz{|}~
```

... but actually, you'll probably only ever really need to think about the first 41. That's because in practice, base quality scores tend to range between 0 and 40 (or `!` to `I`) [^9] [^11]. According to Illumina, a Phred quality score of 40 equals a base call accuracy of 99.99%[^12], so that's pretty solid. 

Putting it all together, here's an example of what a FASTQ file looks like (borrowed from [^13]):

```
@SRR001926.1 FC00002:7:1:111:750 length=36
TTTTTGTAAGGAGGGGGGTCATCAAAATTTGCAAAA
+SRR001926.1 FC00002:7:1:111:750 length=36
IIIIIIIIIIIIIIIIIIIIIIIIIFIIII'IB<IH
```

One thing to note, which is pertinent to this project, is that if you run paired-end sequencing, you'll end up with **two** FASTQ files: a forward read and a reverse read[^9]. 

## Sequence Alignment/Map (SAM) and its variants
**Extension**: `.sam`

After the reads have been mapped to the reference genome (*i.e.*, placing where the reads belong on the genome), the SAM format stores alignment information (*i.e*, how closely the read matches the part of the reference it's been mapped to) for those reads. It is a tab-delimited text file consisting of an optional header section, which stores metadata, and an alignment section, which stores, well, the alignment data[^14] [^15].

To walk us through understanding a SAM file, let's start with an example from the SAM specification that we will break down as we go[^15]:

```
@HD VN:1.6  SO:coordinate
@SQ SN:ref  LN:45
r001   99   ref 7   30  8M2I4M1D3M  =   37  39  TTAGATAAAGGATACTG   *
r002   0    ref 9   30  3S6M1P1I4M  *   0   0   AAAAGATAAGGATA      *
r003   0    ref 9   30  5S6M        *   0   0   GCCTAAGCTAA         *   SA:Z:ref,29,-,6H5M,17,0;
r004   0    ref 16  30  6M14N5M     *   0   0   ATAGCTTCAGC         *
r003   2064 ref 29  17  6H5M        *   0   0   TAGGC               *   SA:Z:ref,9,+,5S6M,30,1;
r001   147  ref 37  30  9M          =   7   -39 CAGCGGCAT           *   NM:i:1
```

Wow, that sure is a lot of stuff! We will know what they mean soon.

### SAM header section
Let's talk about the header section first. Each header line starts with an `@` followed by a two-letter code denoting the record type. The fields following this are identified by two-letter tags and their values. They follow the format `TAG:VALUE`, except for the `@CO` lines, which are free-text comments. Each record type has at least one required field and several optional ones. Here are the record types and the *required* fields for each[^14] [^15]:

- `@HD`: File-level metadata. You can only have one, and it needs to be the first line in the file.
    - `VN`: Format version.
- `@SQ`: Reference sequence. The order of these lines sets the alignment sorting order. 
    - `SN`: Name of the reference sequence. 
    - `LN`: Length of the reference sequence. 
- `@RG`: Read group[^16]. You can have multiple of these in no particular order (unordered). GATK requires a few fields that would otherwise be marked as optional in the SAM specification; I included these.
    - `ID`: Read group identifier. As you might expect, these need to be unique per `@RG` line.
    - `PL`: Platform used to perform the read (*e.g.*, Illumina).
    - `PU`: Platform unit. For Illumina reads, GATK specifies the format `{FLOWCELL_BARCODE}.{LANE}.{SAMPLE_BARCODE}`.
    - `LB`: DNA preparation library identifier. Used to detect potential duplicates.
    - `SM`: Name of the sample.
- `@PG`: Program used.
    - `ID`: Program record identifier. Must be unique.
- `@CO`: One-line, free-text comment. You can use multiple of these, as long as each is preceded by the `@CO` code.

For more information about optional fields, accepted formats, etc. please refer to the SAM specification[^15].

Let's take a closer look at the header of our example SAM file:

```
@HD VN:1.6  SO:coordinate
@SQ SN:ref  LN:45
```

Okay, so we now know that the `@HD` line describes the file itself, and that `VN` tells us the file uses SAM version 1.6. But wait, what is `SO`? Well, that's one of the optional fields, and it refers to the sorting order of alignments. It's set to `coordinate`, meaning the alignments are sorted by the `RNAME` (major key) and `POS` (minor key) fields; we'll get to what those are in a second.

The `@SQ` line describes the reference sequence, which is named `ref` (`SN` field) and has a length of 45 bases (`LN` field).

For funsies, here's another example of what a SAM header might look like (borrowed from [^17]):

```
@HD VN:1.5  GO:none SO:coordinate
@SQ SN:20   LN:63025520 UR:http://www.broadinstitute.org/ftp/pub/seq/references/ref.fasta   M5:0dec9660ec1efaaf33281c0d5ea2560f SP:Homo Sapiens
@RG ID:H0164.2  PL:illumina PU:H0164ALXX140820.2    LB:Solexa-272222    PI:0    DT:2014-08-20T00:00:00-0400 SM:NA12878  CN:BI
```

More optional fields! Here's what they mean:

In the `@HD` line, the `GO` tag refers to the grouping of alignments. It's set to `none` here, meaning the alignments aren't grouped. 

In the `@SQ` line, the `UR` tag is the URI (Uniform Resource Identifier) of the the sequence. In this case, it links directly to the reference FASTA in the Broad Institute's FTP directory. The `M5` tag is the MD5 checksum of the sequence. The `SP` tag is the species of the organism the sequence came from (human, in this case).

In the `@RG` line, the `PI` refers to the predicted median insert size, rounded to the nearest integer. When it's set to 0, as shown here, it most likely means that the predicted median insert size is not known/specified, and not that the library fragments are literally 0 bp long. The `DT` tag is the date and time the run was produced, in ISO 8601 format[^15]. The `CN` tag refers to the name of the sequencing center that produced the read; here, "BI" stands for Broad Institute.

### SAM alignment section
Okay, now for the meat of the matter. In the alignment section, there's one row per aligned read, with 11 mandatory fields for each read. I'll list them first, then talk a little bit more about what some of them mean[^14] [^15] [^18].

1. `QNAME`: Query/read name (FASTQ ID) (set as `*` if unavailable)
2. `FLAG`: Bitwise flags for mapping properties (*e.g.*, paired/mapped/reverse strand/duplicate)
3. `RNAME`: Name of the reference the read is mapped to (same as whatever was specified in the `@SQ SN` tag in the header; `*` if read is unmapped)
4. `POS`: Leftmost mapping position (1-based)
5. `MAPQ`: Mapping quality (calculated as $-10log_{10}\text{Pr}\{\text{mapping position is wrong}\}$, rounded to the nearest integer)
6. `CIGAR`: Alignment representation as a CIGAR string
7. `RNEXT`: Reference name of mate/next read (set as `*` if unavailable, and as `=` if identical to `RNAME`)
8. `PNEXT`: Position of mate/next read (equals `POS` of next read)
9. `TLEN`: Observed template length
10. `SEQ`: Read sequence
11. `QUAL`: Per-base Phred quality (Phred+33)

Again, for more detailed information (input type, optional fields, etc.), look at the SAM specification. Note that all mapped reads will be represented on the **forward genomic strand**, so any read that's been mapped to the reverse strand will be reverse-complemented in the `SEQ` field. Other fields that depend on the sequence (such as `CIGAR` and `QUAL`) will also be reversed to match[^15].

Let's talk about these fields within the context of the example, which contains six aligned reads. For clarity, I'm going to refer to them as Reads 1-6.

```
r001   99   ref 7   30  8M2I4M1D3M  =   37  39  TTAGATAAAGGATACTG   *
r002   0    ref 9   30  3S6M1P1I4M  *   0   0   AAAAGATAAGGATA      *
r003   0    ref 9   30  5S6M        *   0   0   GCCTAAGCTAA         *   SA:Z:ref,29,-,6H5M,17,0;
r004   0    ref 16  30  6M14N5M     *   0   0   ATAGCTTCAGC         *
r003   2064 ref 29  17  6H5M        *   0   0   TAGGC               *   SA:Z:ref,9,+,5S6M,30,1;
r001   147  ref 37  30  9M          =   7   -39 CAGCGGCAT           *   NM:i:1
```

#### `QNAME` field
In the context of a SAM file, "query" is another term for a read that's been mapped and aligned[^15]. It's the first column from the left (all of the fields are in order, by the way), and it shows us that the reads are named `r001`, `r002`, `r003`, and `r004`. What appears confusing at first is that there are two reads named `r001` (Reads 1 and 6) and two reads named `r003` (Reads 3 and 5). Let's see if the other fields can help us figure out why that might be.

#### `FLAG` field
The bitwise flags in the `FLAG` field encodes information about the mapped read. The SAM specification provides a table explaining the bits[^15]:

| Bit         | Description                                                             |
|-------------|-------------------------------------------------------------------------|
| 1       0x1 | template having multiple segments in sequencing                         |
| 2       0x2 | each segment properly aligned according to the aligner                  | 
| 4       0x4 | segment unmapped                                                        |
| 8       0x8 | next segment in the template unmapped                                   |
| 16     0x10 | `SEQ` being reverse-complemented                                        |
| 32     0x20 | `SEQ` of the next segment in the template being reverse-complemented    |
| 64     0x40 | the first segment in the template                                       |
| 128    0x80 | the last segment in the template                                        |
| 256   0x100 | secondary alignment                                                     |
| 512   0x200 | not passing filters, such as platform/vender quality controls           |
| 1024  0x400 | PCR or optical duplicate                                                |
| 2048  0x800 | supplementary alignment                                                 |

So for Read 1, the `FLAG` field has a value of 99. To unpack that, we use bitwise AND operations.

The decimal value 99 in binary is 000001100011. I included the first five zeros to make it easier to decode with the table above. We can see that the following bits are 1:

- 1: the read is paired
- 2: the read is paired properly
- 32: the read's mate is on the reverse strand (the mate is the other read in a read pair)
- 64: it is the first read in the pair

In the bit table, the descriptions are phrased to encompass more than just paired-end reads, but in our case, our template is the original DNA, and the segments are the paired reads (meaning we only have two segments). So, the first segment is the first read, and the next/last segment is the second read (the first read's mate). 

Read 5 in the SAM has a `FLAG` value of 2064. This corresponds to the following bits:

- 16: the read is on the reverse strand
- 2048: it is a supplementary alignment

Bit 2048 indicates that the supplementary alignment is part of a chimeric alignment. The SAM specification states that "All of the SAM records in a chimeric alignment have the same `QNAME`," so we can deduce that Read 3 is also part of the chimeric alignment. Furthermore, since Read 5 has the supplementary alignment flag, that means the Read 3 is the representative alignment.

Finally, let's look at Read 6, which is the last record in the example. It has a `FLAG` value of 147, so these bits are "on":

- 1: the read is paired
- 2: the read is paired properly
- 16: this read is on the reverse strand
- 128: it is the second read in the pair

This information complements what we gleaned from Read 1's entry, telling us that Read 6 is the second read in the `r001` pair. 

From the `FLAG` fields alone, we now understand why there are two `r001` lines and two `r003` lines.

By the way, the Broad Institute has a handy online tool for decoding SAM flags [here](https://broadinstitute.github.io/picard/explain-flags.html), if you're interested in seeing what various flags might look like. 

#### `RNAME` and `RNEXT` fields 
`RNAME` is probably the easiest one to parse: It's just the name of the reference. Note that, for mapped segments, whatever is in this field has to match `@SQ SN`, and indeed, the `RNAME` for all the records is `ref`.

`RNEXT` is the name of the reference of the next read in the pair. Our only pair is `r001`, and both reads use the same reference, so their `RNEXT` fields are `=` to indicate this. For the rest of the reads, `RNEXT` is set as `*` because none of them has a next read.

#### `POS` and `PNEXT` fields
One thing that's important to notice is that `POS` is *1-based* (that is, the first base is counted as 1, not 0—refer to the section about FAI files for a 0-based example). By extension, `PNEXT`, which represents the position of the next read in the template, is also 1-based. As I understand it, it's done this way so that *unmapped* reads can have `POS` set to 0 to indicate that they have no position[^14]. However, doing this means that as you progress through your analysis, there's a potential point of failure if you or your tools don't account for the fact that some file formats are 1-based and others are 0-based. In fact, BAM, which is a compressed variation of SAM, is 0-based[^14] [^15]! Luckily, the tools in a functional pipeline should take care of this shift in coordinate systems for you, but it's helpful to be aware that it exists during debugging or running calculations. This page[^18] explains the "coordinate-system trap" in further detail and how to avoid it.

Notice how the only pair in our example, `r001`, has complementary `PNEXT` values. The `PNEXT` of Read 1 is 37, which is the `POS` of Read 6, and the `PNEXT` of Read 6 is 7, which is the `POS` of Read 1.

In our example, we would position the sequences against the reference like this (the reference is made up in this case):

```
Position:       1234567890123456789012345678901234567890123456789
Reference:      ******TTAGATAAAGGATACTCAGC**TAGGC***CAGCGGCAT
Read 1:         ------TTAGATAAAGGATACT
Read 2:         --------AAAAGATAAGGATA
Read 3:         --------GCCTAAGCTAA
Read 4:         ---------------ATAGCTTCAGC
Read 5:         ----------------------------TAGGC
Read 6:         ------------------------------------CAGCGGCAT
```

Except it wouldn't be like that at all, because of the CIGAR strings. 

#### `CIGAR` field
Now I want to talk about `CIGAR`, because I hadn't heard of this before and the fact that it's called CIGAR fascinated me. Apparently, CIGAR stands for "Compact Idiosyncratic Gapped Alignment Report" and uses a system of operators and numbers to describe how the read aligns to the reference genome[^19]. To illustrate this, let's first look at what the operators are:

| Op    | Description                                               | Consumes query    | Consumes reference    |
|-------|-----------------------------------------------------------|-------------------|-----------------------|
| `M`   | Alignment match                                           | yes               | yes                   |
| `I`   | Insertion (base is absent in reference)                   | yes               | no                    |
| `D`   | Deletion (base is absent in query)                        | no                | yes                   |
| `N`   | Gap/skipped region (multiple bases are absent in query)   | no                | yes                   |
| `S`   | Soft clipping (clipped sequence present in `SEQ`)         | yes               | no                    |
| `H`   | Hard clipping (clipped sequences *not* present in `SEQ`)  | no                | no                    |
| `P`   | Padding (silent deletion from padded reference)           | no                | no                    |
| `=`   | Sequence match                                            | yes               | yes                   |
| `X`   | Sequence mismatch                                         | yes               | yes                   |

In a CIGAR string, every operator is preceded by the number of bases to which it applies. For example, the CIGAR string "4M2I8M" represents a subsequence that contains 4 matching bases, 2 insertions, and 8 more matching bases, in that order. The sum of the length of the `M`, `I`, `S`, `=`, and `X` operations must equal the length of the query sequence[^15].

Several things to unpack here. First, what does it mean for an operation to "consume" the query or the reference? Well, imagine you're the one doing the alignment, on paper, and you have to deal with indels and all that stuff. To keep track of your place in both the query and the reference, you might cross out the bases you've already looked at. Here's a tiny example of what I mean:

You have the reference `AGTCTGA` and the query `AGCTCGA`. You might step through the bases like this:

```
Position:      123456789
Reference:     AGTCTGA
Query:         AGCTCGA

Position 1: Bases match. Consume query and reference.

Reference:     -GTCTGA
Query:         -GCTCGA

Position 2: Bases match. Consume query and reference.

Reference:     --TCTGA
Query:         --CTCGA

Position 3: You happen to know there's an insertion in the query here. Consume query only. 

Reference:     --*TCTGA
Query:         ---TCGA

Positions 4-5: Bases match. Consume query and reference.

Reference:     --*--TGA
Query:         -----GA

Position 6: You happen to know there's a deletion in the query here. Consume reference only.

Reference:     --*---GA
Query:         -----*GA

Positions 7-8: Bases match. Consume query and reference.

Reference:     --*-----
Query:         -----*--
```

It's kind of like that.

Second, what do these operators actually mean? Like, what's the difference between an alignment match (`M`) and a sequence match (`=`)? What is clipping (`S`, `H`) and padding (`P`)?

So, the alignment match operator came before the sequence match and mismatch operators and is a sort of umbrella that encompasses both. That is, an alignment match can be a sequence match or mismatch, because all it means is that the query bases are in the right positions relative to their reference. There's no information about whether those bases actually match the reference or not. `=` and `X` were introduced to add the missing clarity. Now a CIGAR string like "6M" could be rewritten as "4=2X" and you would know that on top of the bases being aligned, the first four bases are exact matches to the reference and the last two bases are mismatches.[^20]

In some cases, the `M` operator is treated as equivalent to `=`, and it is used in combination with `X`. This page shows a few examples of using the operators this way[^19].

Now, let's talk about clipping. Soft- (`S`) and hard-clipping (`H`) are effectively the practice of excluding certain bases from the alignment. The reason for doing this might be that the bases are low-quality read ends, difficult-to-align regions, or repetitive regions, and the alignment would benefit from their exclusion. The difference between soft- and hard-clipping is that soft-clipped bases remain present in the sequence for downstream analysis, while hard-clipped bases are removed from the sequence entirely.[^21]

Padding is a little weird, but its main role appears to be as a placeholder that allows multiple alignments to also align with each other and doesn't consume the query or the reference[^22]. I would probably consider it more of a visual aid than anything, but I might be wrong to interpret it that way. 

Taking CIGARs into consideration, the alignments in our example would actually look more like this:

```
Positions:              12345678901234  5678901234567890123456789012345
Reference:              AGCATGTTAGATAA**GATAGCTGTGCTAGTAGGCAGTCAGCGCCAT
Read 1:                 ------TTAGATAAAGGATA*CTG
Read 2:                 -----aaaAGATAA*GGATA
Read 3:                 ---gcctaAGCTAA
Read 4:                 -----------------ATAGCT..............TCAGC
Read 5:                 ------------------------ttagctTAGGC
Read 6:                 --------------------------------------CAGCGGCAT
```
The asterisks (`*`) represent padding and deletions, the periods (`.`) represent gaps, and lowercase bases represent soft-clipped regions. And yes, I did happen to find the actual reference sequence from the original paper introducing the SAM format[^22].

#### `MAPQ` field
Okay, I know this is out of order because this should have gone before the `CIGAR` section, but the segue was just too smooth and I couldn't bring myself to interrupt that flow. But we're talking about it now, and that's what counts.

The `MAPQ` field is a quality score of the mapping. It is calculated by $-10log_{10}\text{Pr}\{\text{mapping position is wrong}\}$, rounded to the nearest integer. Most of the `MAPQ` scores in our example are 30, so after we do a little algebra, we find that the probability that the mapping position is wrong is 1/1000, or 0.0001. That seems pretty good to me. 

Read 5 has a `MAPQ` of 17, which is equivalent to a probability of the mapping position being wrong of about 0.01995. That is not as good. Kinda makes sense why this is the supplementary alignment.

#### `TLEN` field
This is another field that mostly applies to paired-end reads, which is made evident by the fact that again, in our example, the only non-zero `TLEN` values are associated with `r001`. The template length is the signed distance between the first mapped base of the first read and the last mapped base of the second read, inclusively ($\text{end} - \text{start} + 1$)[^15]. The sign indicates which segment is the leftmost (positive) and which is the rightmost (negative). 

Let's calculate this. The position of the last mapped base of Read 6 is 45 (see alignment above), and the position of the first mapped base of Read 1 is 7. 45 - 7 + 1 = 39. Therefore, the template, which spans the two reads and the distance between them, has a length of 39. We can confirm this by seeing that the `TLEN` values for Read 1 and Read 6 are indeed 39 and -39, respectively. 

#### `SEQ` field
I'm including this for completion's sake, but I think this field is fairly self-explanatory. It's the segment sequence itself, and we've already been using the sequences from the example.

#### `QUAL` field
The `QUAL` field basically contains the Phred+33 quality sequence from the FASTQ the read originated from. When there is no quality score, it's set as `*` instead, as seen in our example.

#### (Some of) the optional fields
Okay gang, home stretch. The alignment section has optional fields, and I simply will not cover all of them (read about them yourself here[^23] if you want), but I'll talk briefly about the ones shown in the example.

The optional fields follow the format `TAG:TYPE:VALUE`, wherein `TAG` is a two-character string similar to the tags in the header section, and`TYPE` is a single case-sensitive letter denoting the format of `VALUE`[^15].

`SA:Z` describes "Other canonical alignments in a chimeric alignment," and is appropriately assigned to Reads 3 and 5, which we discovered earlier were parts of a chimeric alignment. It includes the `rname`, `pos`, `strand`, `CIGAR`, `mapQ`, and `NM` fields, where `NM` is the number of mismatches[^23]. In the example, the `SA:Z` fields of Read 3 and 5 contain each other's information, which seems to suggest the assignment of the alignment information to a particular read was arbitrary and could have just as well been swapped without making a meaningful difference.

`NM:i`, as mentioned, denotes the number of mismatches. This includes insertions and deletions[^23]. Thus, we know that in Read 6, there is one mismatch. Knowing that the region the read mapped to in the reference is `CAGCGCCAT`, this could have also been expressed in `CIGAR` as "5=1X3=", which would have also told us where the mismatch was. 

Okay, I think that's everything I want to cover with SAM format. That sure was a lot. Thanks for hanging in there.

### Sequence dictionary (DICT)
**Extension**: `.dict` (appended to the end of a FASTA filename, replacing the original extension)

Now that we understand what SAM files are, let's talk about sequence dictionaries real quick. A sequence dictionary is a file built like a SAM header containing metadata about the reference FASTA genome, including reference sequence/contig names, lengths, and MD5 checksums[^24] [^25]. It is generated from the reference FASTA and used to run compability checks throughout the analysis pipeline. Like the FAI file, it is required by GATK workflows as part of the "prepared reference triad"[^25]. The FASTA is the reference genome, the FAI is the index for finding contigs wtihin the genome, and the DICT is the dictionary containing information about what the contigs actually are.

The file consists of a header line pretty much exactly like a SAM `@HD` line and one `@SQ` line for every contig. The `@SQ` lines contain the following fields: `SN`, `LN`, `M5`, and `UR`. As it happens, we are already familiar with these fields, so I won't re-explain them. If you're skipping around as you read this, then refer to the section above ([SAM files](#sequence-alignmentmap-sam-and-its-variants)).

Here's a small example from the GATK Team [^5]:

```
@HD VN:1.5
@SQ SN:20   LN:63025520 M5:0dec9660ec1efaaf33281c0d5ea2560f UR:file:/Users/vdauwera/Desktop/germline_mini/ref/ref.fasta
```

### Binary Alignment/Map (BAM)
**Extension**: `.bam`

Perhaps the most important thing to know about BAM is that **it is the primary format used to store alignment data** in nearly all modern NGS workflows[^17] [^27]. That means even if your aligner produces a SAM, it will likely soon be converted to BAM, either automatically or as necessitated by downstream tools. It's not too different from a SAM, though, and is used more or less the same way. In fact, a BAM file contains all the same information as a SAM file, but encoded in binary and compressed in the BGZF format (you'll learn about this [later](file_formats.md#blocked-gnu-zip-format-bgzf)). This makes the file "smaller and indexable, but not human-readable"[^26]. The benefits and ubiquity of BAM mean the user has much to gain and little to lose by using BAM over SAM. I mean, how likely is it that you were *really* planning to read SAM files with your human eyes?

As I mentioned earlier, BAM is 0-based rather than 1-based, contrary to SAM; this is usually handled automatically when you are using tools, but you should still be aware of it in case you ever need to switch to manual processing[^26] [^27]. It also stores multi-byte numbers in little-endian order, which may or may not become relevant. If you enjoy learning about details like that, the SAM specification includes sections explaining the BAM format that you may find interesting[^15].

While BAM is typically used to store *aligned* reads, there is a variant called **uBAM** (unmapped BAM) that stores *unaligned* reads. It's used because it can store read-associated metadata that would otherwise be lost in the FASTQ[^27] [^28]. This is what GATK workflows start with instead of FASTQs[^28], but I'll talk more about that elsewhere.

There are two formats that are used to index a BAM file: the BAM Index (BAI; extension `.bai`) and the Coordinated-Sorted Index (CSI; extension `.csi`). Both use a binning scheme to build the index[^15], but CSI offers configurable bin parameters to overcome BAI's sequence length limitation[^29] [^30]. In general, BAI is more established and better supported than CSI because it has been around for longer, so most people don't bother with CSI unless their specific use case requires it [^29] [^30]. Either way, you'll still want an index for your BAM file for the same reason you want an index for your FASTA file (GATK requires it)[^17].

Another compressed version of SAM is CRAM (Compressed Reference-oriented Alignment/Map), which is mostly used for archival purposes[^17] and won't be covered here.

## Variant Call Format (VCF)
**Extension**: `.vcf`

The VCF is where your called variants go, as you might have deduced from the name Variant Call Format. It's a tab-delimited text file that stores SNPs, indels, and structural variants, along with their associated information [^31] [^32]. It comprises three sections: the meta-information lines, a header line, and data lines[^32] [^33]. Like with the SAM format, I'll break down each component within these sections with an example. 

**By the way**: Because VCFs can get large and unwieldy, an alternative exists called Binary Call Format (BCF) which, like BAM, is encoded in binary, compressed in BGZF format, produces smaller files, and is not human-readable. Its use is desirable in situations where performance is key and human-readability is not a concern[^34]. It is also worth noting at this point that the Broad Institute uses an extended version of the VCF format called gVCF, or genomic VCF[^35], but I won't cover that here.

Okay, let's take a look at the example I will be using to explore VCF. I borrowed this from the official VCF specification[^33]:

```
##fileformat=VCFv4.5
##fileDate=20090805
##source=myImputationProgramV3.1
##reference=file:///seq/references/1000GenomesPilot-NCBI36.fasta
##contig=<ID=20,length=62435964,assembly=B36,md5=f126cdf8a6e0c7f379d618ff66beb2da,species="Homo sapiens",taxonomy=x>
##phasing=partial
##INFO=<ID=NS,Number=1,Type=Integer,Description="Number of Samples With Data">
##INFO=<ID=DP,Number=1,Type=Integer,Description="Total Depth">
##INFO=<ID=AF,Number=A,Type=Float,Description="Allele Frequency">
##INFO=<ID=AA,Number=1,Type=String,Description="Ancestral Allele">
##INFO=<ID=DB,Number=0,Type=Flag,Description="dbSNP membership, build 129">
##INFO=<ID=H2,Number=0,Type=Flag,Description="HapMap2 membership">
##FILTER=<ID=q10,Description="Quality below 10">
##FILTER=<ID=s50,Description="Less than 50% of samples have data">
##FORMAT=<ID=GT,Number=1,Type=String,Description="Genotype">
##FORMAT=<ID=GQ,Number=1,Type=Integer,Description="Genotype Quality">
##FORMAT=<ID=DP,Number=1,Type=Integer,Description="Read Depth">
##FORMAT=<ID=HQ,Number=2,Type=Integer,Description="Haplotype Quality">
#CHROM  POS     ID          REF ALT     QUAL FILTER INFO                                FORMAT      NA00001         NA00002         NA00003
20      14370   rs6054257   G   A       29   PASS   NS=3;DP=14;AF=0.5;DB;H2             GT:GQ:DP:HQ 0|0:48:1:51,51  1|0:48:8:51,51  1/1:43:5:.,.
20      17330   .           T   A       3    q10    NS=3;DP=11;AF=0.017                 GT:GQ:DP:HQ 0|0:49:3:58,50  0|1:3:5:65,3    0/0:41:3
20      1110696 rs6040355   A   G,T     67   PASS   NS=2;DP=10;AF=0.333,0.667;AA=T;DB   GT:GQ:DP:HQ 1|2:21:6:23,27  2|1:2:0:18,2    2/2:35:4
20      1230237 .           T   .       47   PASS   NS=3;DP=13;AA=T                     GT:GQ:DP:HQ 0|0:54:7:56,60  0|0:48:4:51,51  0/0:61:2
20      1234567 microsat1   GTC G,GTCT  50   PASS   NS=3;DP=9;AA=G                      GT:GQ:DP    0/1:35:4        0/2:17:2        1/1:40:3
```

### Meta-information lines
The meta-information lines are each preceded by two pound signs (`##`) and are either unstructured or structured[^33]. They are listed in alphabetical order[^31].

*Unstructured* meta-information lines are formatted as a key-value pair, like this: `##key=value`. They typically communicate information about things like how the file was generated, contig declarations, phasing, etc. Save for the `fileformat` line[^33], these do not appear to be mandatory, but I reckon it's good practice to include at least some of them if that information is available.

*Structured* meta-information lines are also key-value pairs, but the values are *also* key-value pairs: `##KEY=<key=value,key=value,key=value,...>`. (Note the lack of spaces.) They have to do with the fields in the header line; in the example, you can see that all of the structured meta-information lines start with `INFO`, `FILTER`, or `FORMAT`, which are all column names. All structured meta-information lines require an `ID` field (unique within all lines with the same `##key=`) and a `Description` field. The `INFO` and `FORMAT` lines additionally require a `Number` field and a `Type` field for each line. There are also some optional fields for some of the lines[^33]. 

These lines are helpful because they explain what certain pieces of information actually mean in the data records. For instance, now that we've read the `INFO` meta-information lines, we know how to parse `NS=3;DP=14;AF=0.5;DB;H2` under the `INFO` column in the first record of our example. There are three samples containing this variant; the variant has a depth of 14 and an allele frequency of 0.5; the variant is present in both the dbSNP (Build 129) and HapMap 2 databases. 

GATK has a special structured meta-information line called `GATKCommandLine` that contains the parameters used by GATK to produce the VCF[^31]. Here's an example of what that looks like:

```
##GATKCommandLine.HaplotypeCaller=<ID=HaplotypeCaller,Version=3.7-0-gcfedb67,Date="Fri Jan 20 11:14:15 EST 2017",Epoch=1484928855435,CommandLineOptions="[command-line goes here]">
##GATKCommandLine=<ID=GenotypeGVCFs,CommandLine="[command-line goes here]",Version=4.beta.6-117-g4588584-SNAPSHOT,Date="December 23, 2017 5:45:56 PM EST">
```

### Header line and data lines
I combined the header and data into one section because I figured it would make more sense to explain them together. In fact, some people[^31] prefer to consider VCFs as having two main parts rather than three, and I think that's valid. 

The header line is basically the same as a header row on a table. There are eight mandatory (or fixed) fields, but many VCF files include a ninth. These fields are, in order[^31] [^33]:

1. `CHROM`: The chromosome on which the variant occurs.
2. `POS`: The position at which the variant occurs, where the first base has position 1 (1-based coordinate system).
3. `ID`: The unique identifier(s) of the variant. If the variant is in the dbSNP database, use the rs number(s). If not available, use `.`.
4. `REF`: The reference base or sequence. Each base must be A, C, G, T, or N; if the sequence has IUPAC ambiguity codes, reduce the ambiguous bases to whichever options come first alphabetically (e.g., R = A/G --> A).
5. `ALT`: Alternate base(s), with the same base constraints as `REF`. If a deletion, use `*`. If no variant, use `.`.
6. `QUAL`:  Phred-scaled quality score for `ALT` value. In other words, the probability that a `REF`/`ALT` polymorphism does indeed exist at this site.
7. `FILTER`: `PASS` if the variant call passes all filters; otherwise, list the filters that the record failed, separated with a semicolon (`;`).
8. `INFO`: Additional information about the variant, with each piece of information separated by semicolons (`;`). Formatted as a series of keys, some of which might have values. If no information is available, use `.`. A table containing reserved `INFO` keys is available in the VCF specification.
9. `FORMAT`: Specifies the format of the information contained in the genotype/sample fields. Optional.

The above fields are then followed by any number of sample- or genotype-level fields that provide information specific to that sample/genotype. In our example, there are three such fields: `NA00001`, `NA00002`, and `NA00003`. In order to know how to interpret the information in the genotype columns, we must look to the `FORMAT` column, as well as the meta-information lines associated with that field.

```
##FORMAT=<ID=GT,Number=1,Type=String,Description="Genotype">
##FORMAT=<ID=GQ,Number=1,Type=Integer,Description="Genotype Quality">
##FORMAT=<ID=DP,Number=1,Type=Integer,Description="Read Depth">
##FORMAT=<ID=HQ,Number=2,Type=Integer,Description="Haplotype Quality">
#CHROM  POS     ID          REF ALT     QUAL FILTER INFO                                FORMAT      NA00001         NA00002         NA00003
20      14370   rs6054257   G   A       29   PASS   NS=3;DP=14;AF=0.5;DB;H2             GT:GQ:DP:HQ 0|0:48:1:51,51  1|0:48:8:51,51  1/1:43:5:.,.
20      17330   .           T   A       3    q10    NS=3;DP=11;AF=0.017                 GT:GQ:DP:HQ 0|0:49:3:58,50  0|1:3:5:65,3    0/0:41:3
20      1110696 rs6040355   A   G,T     67   PASS   NS=2;DP=10;AF=0.333,0.667;AA=T;DB   GT:GQ:DP:HQ 1|2:21:6:23,27  2|1:2:0:18,2    2/2:35:4
20      1230237 .           T   .       47   PASS   NS=3;DP=13;AA=T                     GT:GQ:DP:HQ 0|0:54:7:56,60  0|0:48:4:51,51  0/0:61:2
20      1234567 microsat1   GTC G,GTCT  50   PASS   NS=3;DP=9;AA=G                      GT:GQ:DP    0/1:35:4        0/2:17:2        1/1:40:3
```

Here we see that genotype information is formatted as a series of data separated by colons. The data types are represented by keywords in the `FORMAT` column, the meanings of which are explained by the meta-information lines above. Thus, we learn that `GT` represents genotype as one string, `GQ` represents genotype quality (Phred-scaled) as one integer, `DP` represents read depth as one integer, and `HQ` represents haplotype quality (Phred-scaled) as two integers (separated by a comma; missing values are represented with `.`). Like with `INFO`, the VCF specification contains a table of reserved keys for `FORMAT`.

I think we get the idea with `GQ` and `DP`, but `GT` might be harder to parse. The strings used to represent genotype are formatted as two numbers, separated by either a vertical bar (`|`) or a forward slash (`/`). These have to do with whether the allele is phased or unphased, the ploidy of the sample, and which `ALT` allele was detected (or whether it's the `REF` allele)[^33]. They break down like this:

- The fact that every `GT` value is comprised of two numbers (`0|0`, `1|2`, `0/1`, etc.) shows that the samples (and by extension, the reference) came from diploid organisms; a haploid sample would contain only one number, a triploid would contain three, and so on. 
- `|` means the allele is phased, and `/` means the allele is unphased. Phasing indicates whether we know which haplotype the alleles belong to.
- `0` means the allele matches the reference. `1` means the allele matches the first alternate, and `2` means the allele matches the second alternate, and so on. 

Putting this together, let's look at the `GT` of `NA00002` for the variant `rs6054257` (first row). The value is `1|0`, so we know that for this variant, the sample is heterozygous and carries a copy of the `ALT` allele and a copy of the `REF` allele, respectively. We also know that it's phased. 

Knowing about phasing also explains why some samples are missing `HQ` values. Haplotype quality refers to our level of confidence that we correctly assigned the alleles to their haplotypes. Now notice that only the *unphased* calls lack an `HQ` value. If we don't know which haplotypes the alleles came from to begin with, we can't calculate a haplotype quality. 

The GATK team walks through their own example with different `FORMAT` keywords, so I think it'll be helpful to look at their article, too[^31]. 

## Browser Extensible Data (BED)
**Extension**: `.bed`

BED is a relatively uncomplicated format; it simply lists genomic features and their positions as a whitespace-delimited (usually tab, `\t`) text file[^36] [^37]. At the broadest level, they store genomic regions of interest that might then be sequenced, analyzed, or visualized[^37]. The data lines consist of at least three fields, listed below[^36]:

1. `chrom`: Chromosome name.
2. `chromStart`: Feature start position.
3. `chromEnd`: Feature end position.

It's worth noting here that BED uses what is called a 0-based, half-open coordinate system. This means that the coordinate of the very first base is 0, and the end coordinate is just past the last base in the interval. This is contrary to other file formats, most of which use a 1-based, inclusive coordinate system, and so can cause problems if you don't remember to convert coordinates when working between formats. Refer again to this page[^18] discussing the "coordinate system trap" and how to avoid it.

There are nine more optional fields as well. Fields 4-6 are for basic information, while fields 7-12 pertain to visualization.

4. `name`: Feature description.
5. `score`: A number between 0 and 1000. What the number actually means depends on context, and it can be 0 across all data lines if the field is included but not used.*
6. `strand`: Strand feature is located on, represented as `+` (coding), `-` (complementary), or `.` (none).
7. `thickStart`: Start position of a region that will be visualized in a bold/accented display. Equal to `chromStart` if included but not used.
8. `thickEnd`: End position of a region that will be visualized in a bold/accented display. Equal to `chromEnd` if included but not used.
9. `itemRgb`: RGB value of the color used to visualize a region or feature. `0` if included but not used.
10. `blockCount`: Number of blocks in the feature. Can be used to represent exons, UTRs, etc.[^37] If this field is used, `blockSizes` and `blockStarts` must also be used.
11. `blockSizes`: List of sizes for each block, separated by commas without spaces.
12. `blockStarts` List of start positions for each block, separated by commas without spaces. Blocks must be contained within the feature and cannot overlap.

* The reason this would be the case is because BED files don't include header lines. Therefore, the fields need to be listed in order to be recognized properly; you can't skip fields. Similarly, if a field is used in any data line, it must be used in *every* data line. 

You can also define custom fields, which are subject to the same constraints as standard fields.

Since BED fields can contain a variable number of fields, they are sometimes referred to with numbers specifying how many fields they contain. For example, BED3 would contain the first three fields, while BED12 would contain all 12 fields. Also, BED6+ files would contain *at least* the first six fields, while a BED6+4 contains the first six fields, followed by four custom fields.

Here's an example of a BED12 file from the UCSC Genome Browser FAQ, as shown in the BED specification[^36]:

```
chr22 1000 5000 cloneA 960 + 1000 5000 0 2 567,488, 0,3512
chr22 2000 6000 cloneB 900 - 2000 6000 0 2 433,399, 0,3601
```

## Blocked GNU Zip Format (BGZF)
**Extension**: `.gz` (appended to original filename, including the original extension)

Finally, you should know about compressed files, also referred to as zipped files. In bioinformatics, the standard compression scheme is BGZF, which is block-based[^15] and built on top of `gzip` (or GNU Zip), the compressed file format developed by the GNU Project[^38]. While BGZF and related formats *are* file formats, a `.gz` extension doesn't say what the file is so much as what's been *done* to it. A compressed file contains all of the same data as the original file (usually), except it's been squished down to reduce the size of the file—this is really important when you are working with large files! There's a whole world of different compression algorithms out there, but that's not the focus of this project, so instead I'll talk briefly about how a BGZF file works.

A BGZF file is made up of blocks, which are themselves gzip archives. (This means they can be decompressed with the same tools you would use to unzip plain gzipped files.) Each block is at most 64 KiB in size and contains an extra field denoting the total block size. When decompressed, the blocks concatenate to produce the original data. The most significant feature of this setup is that it supports random access via virtual file offsets, which are stored by index files like BAI (for BAM) and CSI/TBI. Each offset actually contains *two* offsets: one offset points to the beginning of the BGZF block within the BGZF file, and another offset points to the uncompressed data represented by the same block.[^15] This webpage[^39] contains a simple diagram that illustrates the blocks and offsets, which might help solidify the concept.

A BGZF file also contains an end-of-file (EOF) marker, which is an empty block located at—you guessed it—the end of the file. If your BGZF file doesn't end with this block, it's possible that something went wrong and the file was truncated. One should note, however, that the presence of an EOF marker block by itself doesn't mean much; it needs to be specifically at the end of the file. There are cases when data are appended to existing files without removing the first EOF marker, resulting in a file with two EOF markers: one that has effectively become just a random empty block somewhere in the file, and another at the new end of the file that's the actual EOF marker. So basically, to ensure your file is complete, you need to check two things: that a) an EOF marker block exists, and b) that marker block is at the end of the file.[^15]

## Sources
[^1]: https://fasta.bioch.virginia.edu/fasta_www2/fasta_list2.shtml
[^2]: https://www.bioinformatics.org/sms/iupac.html
[^3]: https://blast.ncbi.nlm.nih.gov/doc/blast-topics/
[^4]: https://en.wikipedia.org/wiki/FASTA_format#NCBI_identifiers
[^5]: https://gatk.broadinstitute.org/hc/en-us/articles/360035531652-FASTA-Reference-genome-format
[^6]: https://en.wikipedia.org/wiki/FASTA_format#Filename_extension
[^7]: https://bffo.org/format/FAI/ 
[^8]: https://www.htslib.org/doc/faidx.html
[^9]: https://bffo.org/format/FASTQ/
[^10]: https://en.wikipedia.org/wiki/Phred_quality_score
[^11]: https://en.wikipedia.org/wiki/FASTQ_format#Encoding
[^12]: https://www.illumina.com/Documents/products/technotes/technote_Q-Scores.pdf
[^13]: https://resources.qiagenbioinformatics.com/manuals/clcgenomicsworkbench/current/index.php?manual=Quality_scores_in_Illumina_platform.html
[^14]: https://bffo.org/format/SAM/
[^15]: https://samtools.github.io/hts-specs/SAMv1.pdf
[^16]: https://gatk.broadinstitute.org/hc/en-us/articles/360035890671-Read-groups
[^17]: https://gatk.broadinstitute.org/hc/en-us/articles/360035890791-SAM-or-BAM-or-CRAM-Mapped-sequence-data-formats
[^18]: https://www.datanovia.com/learn/bioinformatics/foundations/bioinformatics-file-formats#the-coordinate-system-trap-read-this-twice
[^19]: https://replicongenetics.com/cigar-strings-explained/
[^20]: https://omicstutorials.com/step-by-step-guide-understanding-and-redefining-the-cigar-string-in-sam-bam-format/
[^21]: https://omicstutorials.com/step-by-step-guide-to-understanding-soft-clipped-and-hard-clipped-reads-in-sam-bam-files/
[^22]: https://academic.oup.com/bioinformatics/article/25/16/2078/204688?login=false 
[^23]: https://samtools.github.io/hts-specs/SAMtags.pdf
[^24]: https://gatk.broadinstitute.org/hc/en-us/articles/360036712531-CreateSequenceDictionary-Picard
[^25]: https://bffo.org/format/Sequence-Dictionary/
[^26]: https://www.datanovia.com/learn/bioinformatics/foundations/bioinformatics-file-formats#sambam-aligned-reads
[^27]: https://bffo.org/format/BAM/
[^28]: https://gatk.broadinstitute.org/hc/en-us/articles/360035532132-uBAM-Unmapped-BAM-Format
[^29]: https://bffo.org/format/BAI/
[^30]: https://bffo.org/format/CSI/
[^31]: https://gatk.broadinstitute.org/hc/en-us/articles/360035531692-VCF-Variant-Call-Format
[^32]: https://bffo.org/format/VCF/
[^33]: https://samtools.github.io/hts-specs/VCFv4.5.pdf
[^34]: https://bffo.org/format/BCF/
[^35]: https://gatk.broadinstitute.org/hc/en-us/articles/360035531812-GVCF-Genomic-Variant-Call-Format
[^36]: https://samtools.github.io/hts-specs/BEDv1.pdf
[^37]: https://bffo.org/format/BED/
[^38]: https://en.wikipedia.org/wiki/Gzip 
[^39]: https://learngenomics.dev/docs/genomic-file-formats/compression-and-BGZF/