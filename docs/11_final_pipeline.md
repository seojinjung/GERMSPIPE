# The GERMSPIPE pipeline
(draft)

## Phase 0: All the data
File formats: FASTA --> FAI, DICT

Tool(s): Picard `CreateSequenceDictionary`, SAMtools `faidx` (maybe)



## Phase 1: Read pre-processing (optional)
File format: FASTQ (raw) --> FASTQ (processed)

Tool(s): fastp

### Step 1.1: Adapter trimming
Parameters:
    - `--detect_adapter_for_pe` (auto-detection of adapters)

### Step 1.2: Quality trimming

### Step 1.3: Short read filtering

## Phase 2: Alignment
File format: FASTQ --> BAM

Tool(s): minibwa, 

### Step 2.1: Mapping to reference

### Step 2.2: Marking duplicates

### Step 2.3: Base Quality Score Recalibration

### Step 2.4: Hybrid selection metrics

## Phase 3: Variant discovery 

### Step 3.1: Variant calling

### Step 3.2: Statistical filtering

### Step 3.3: Variant annotation

Phase 4: Data visualization