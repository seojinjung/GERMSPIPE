# The GERMSPIPE pipeline
Now that I've figured out the actual software components of my pipeline, I put together a command-line version of my workflow. Put simply, it's just a list of all the commands I'm using to get from FASTQ to VCF/BED. 

[it's all coming together. B)]

## Phase 0: Construct reference genome triad
File formats: FASTA --> FAI, DICT

Tools: Picard `CreateSequenceDictionary`, SAMtools `faidx` (maybe)

Inputs: `GRCh38-GIABv3.fasta.gz`

Arguments:

- 

Outputs: `GRCh38-GIABv3.fasta.gz`, `GRCh38-GIABv3.fasta.gz.fai`, `GRCh38-GIABv3.dict`

## Phase 1: Read pre-processing (optional)
File format: FASTQ (raw) --> FASTQ (processed)

Tool(s): fastp

Input(s):

Arguments:
    
- Adapter trimming: `fastp --detect_adapter_for_pe` (auto-detection of adapters)
- Quality trimming: 
- Short read filtering:

Output(s):

## Phase 2: Alignment
File format: FASTQ --> BAM

Tool(s): minibwa, samtools, FastDup, Picard `CollectHsMetrics`

Input(s):

Arguments:

- Map to reference:
- Convert to BAM: 
- Sort BAM by coordinates: 
- Mark duplicates:
- HS metrics:

Output(s):

## Phase 3: Variant discovery 
File format: BAM --> VCF

Tool(s):

Input(s):

Arguments:

- Variant calling:
- Statistical filtering:
- Variant annotation:

Output(s)

## Phase 4: Data visualization

# Testing the pipeline - does it work?

## Test 1: One at a time

## Test 2: Altogether