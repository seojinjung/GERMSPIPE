# Pre-processing: Getting the reads ready
File format: FASTQ (raw) --> FASTQ (improved)

In this phase, I will send the sample data through some pre-processing steps to get them ready for analysis. Most pre-processing tools are multifunctional and are able to perform adapter trimming, quality trimming, and read filtering, which I think is great for keeping things simple. As a reminder, here are what those steps are meant to accomplish:

- **adapter trimming**: getting rid of leftover adapters from the sequencing process
- **quality trimming**: removing low-quality read ends
- **read filtering**: remove reads that are too short and might get misaligned

I decided to compare **fastp** (Chen 2025) and **Trimmomatic** (Beier *et al.* 2026) as candidates for the pre-processing step of my pipeline. Trimmomatic has been on the scene for a very long time and is pretty much the go-to pre-processing tool in various applications of bioinformatics. fastp is newer by comparison but seems popular in its own right. The representative papers for each include some comparisons to each other, though the exact details of the comparisons differ. I also think it should be a given that each tool wants to hype itself up, so I'm approaching the results with some skepticism. That said, here are some things of note:

- fastp's representative paper was published in 2025, and compared fastp v1.0 with Trimmomatic v0.39. Trimmomatic's latest representative paper was published in 2026 and compared Trimmomatic v0.40 with fastp v1.1.0 and fastp v1.3.3.
- fastp boasts more features than Trimmomatic, including detailed and interactive QC reports (in HTML and JSON), auto-detection of adapters, and, purportedly, a user-friendly interface.
- Trimmomatic is optimized for Illumina data and recently expanded its trimming and filtering toolkit, though the new features seem, at a glance, comparable to what is included in fastp.
- In both speed comparisons, fastp appears to be faster than Trimmomatic when configured with four threads, but Trimmomatic eventually outperforms fastp past eight threads. fastp used five single-end FASTQ files for its comparison, while Trimmomatic used one paired-end FASTQ (accession ID SRR2052337). fastp also performed more analysis operations than Trimmomatic in its comparison; Trimmomatic probably only compared the operations both tools can perform. 
- Trimmomatic's analysis showed that fastp consistently used less memory across threads and versions, and regardless of whether fastp was provided an explicit adapter sequence or detected it by itself.
- In Trimmomatic's evaluation of compressed output file sizes, it produced smaller compressed files than fastp.
- In fastp's evaluation of post-filtering file sizes (FASTQ was gzipped before and after pre-processing), it noted that Trimmomatic produced good-quality data but dropped a lot of data. 

I would refer to another research team for a slightly more objective comparison, but I haven't found any up-to-date papers comparing the latest versions of these tools in a human WES context. The thing to do instead would be to perform my own comparison, but because the focus of my project is on variant discovery and not pre-processing tools, I decided to just pick one based on the information I already have access to and move on. If you find any interesting papers on this, feel free to let me know. In fact, that goes for any component of the pipeline, because I'm sure there are plenty of papers that slipped under my radar while I was researching. 

In the end, I decided to go with **fastp** because it's modern, user-friendly, and seems faster and lighter than Trimmomatic under the circumstances that are relevant to me. fastp is also highly multifunctional, possessing QC capabilities that it uses to produce HTML reports before and after pre-processing, ostensibly eliminating the need for a separate QC tool like FastQC. Furthermore, since fastp can auto-detect common adapter sequences, and the samples I'm using are from standardized datasets, I think it's a fairly safe bet that I won't run into any issues using this feature. In fact, I think it's probably better than running the risk of supplying fastp with the wrong adapters.

**Tool used in this step: fastp**

## Plot twist: Trimming might not be necessary?
While researching trimming tools, I came across a paper from 2024 by Barbitoff and Predeus claiming that read trimming is not necessary for germline short variant calling because it has, purportedly, little to no impact on the quality of the results. In fact, it might even be detrimental to some degree. The authors bring into question the true necessity of what has been a long-standing step in a standard NGS analysis protocol. Now, the experiment was run on both WGS and WES datasets, and for high-coverage WES data, some improvement *was* observed after trimming adapters. 

In any case, I figured this was somewhere I could do a little experimenting on my own. I'll run my pipeline in three different ways:

1. With all pre-processing steps (adapter trimming, low-quality trimming, short read filtering)
2. Without adapter trimming, but still doing the other pre-processings steps
3. No pre-processing at all

Then I'll compare the results from each run. Across three WES samples, that's already nine runs ... hoo, boy. We'll see how it goes.

I also considered what parameters I should use for quality trimming and short read filtering. At this time, I'll go with fastp's defaults, if there are any. If not, I'll look into it a bit more.