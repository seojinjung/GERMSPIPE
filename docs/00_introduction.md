# Introduction to GERMSPIPE: What is going on here?
Recap from the README description: I wanted to sequence my genome, realized I kinda can't right now, and researched and replicated a germline short variant discovery pipeline to cope.

The purpose of these Markdown files is to walk you through my process of learning about the components of the pipeline and reimplementing them. The emphasis is on learning and not on outputting a polished product; if you want to use a pipeline for actual bioinformatics research, you're better off using the ones I referenced, like any of the GATK-based pipelines and nf-core/sarek. I am building a simplified version, the purpose of which is to teach myself about what is inside a bioinformatics pipeline, how it works, and why it works that way.

Okay, but what actually *is* a germline short variant discovery pipeline?

## The name
I think breaking down the project name will give us a better sense of the scope and aim of what I'm actually trying to accomplish here. I am assuming a baseline knowledge of biology going forward, mostly because I kind of doubt anyone who *doesn't* have a biology background is reading this.

A **germline** is the population of germ cells, which give rise to gametes[^1]. In this context, they are where *hereditary* mutations, or variants, occur[^2]. You have these from the moment you're born (well, technically from the moment you exist as a zygote). Contrary to this are *somatic* variants, which are acquired after you're born; these are usually talked about in reference to cancerous mutations[^3].

A variant is anywhere the DNA differs from a reference genome. These are your indels, your SNPs, your extra/missing copies, etc. You've probably heard of genetic mutations; "variants" is a nicer term for the same thing. A **short variant** is a small alteration in the DNA sequence: the SNPs (single-nucleotide polymorphisms) and indels (insertions/deletions)[^4]. Larger mutations are called *structural variants*, and they typically refer to alterations that span more than 50 base pairs[^5].

**Discovery**, in this context, is the process of *finding* variants in the sequencing data, as opposed to simply annotating variants we already know exist. I plan on annotating, too, though.

A **pipeline** is just the pre-packaged workflow that strings together the steps involved in analyzing the data into, ideally, a single command. 

Finally, I mentioned in the README that I intend to analyze **whole exomes**. An exome is the part of the genome that's all exons[^6]; that is, the parts of a gene that make it to the final RNA after splicing.

Put together, this project is a *series of processing elements designed to find inherited indels and SNPs in a person's exons*. So why did I chose these particular characteristics? Well, feasibility, mostly. I was originally going to do the whole genome, but exomes are much smaller (something like 1.5% percent of the human genome[^6] [^7]) while still having 85% of disease-related mutations[^8], which is great news for me as someone who's trying to run this on a laptop. Also, getting my exome sequenced is cheaper than doing my whole genome. Somatic variant calling is usually done in the context of cancer and I don't have cancer; cancer is also famously messy stuff. Germline variant calling strikes me as cleaner. Finally, I see calling short variants as a somewhat simpler starting point, rather than trying to detect *all* variants at once. Basically, these decisions were made to set me up for a smoother learning experience that I could see through to completion without having to worry too much about whether I have enough processing power. I also fully intend to get myself sequenced someday, so there's that, too.

As for why it's called GERMSPIPE, if you work with scientific software literally at all, particularly in the realm of computer science, you've probably run into packages and tools and algorithms named with silly little acronyms that are very clearly the results of a lot of stretching and shoehorning because the researchers really wanted it to be that name no matter what kind of alphanumeric gymnastics they had to perform to make it work. (Example: Illumina's DRAGEN stands for Dynamic Read Analysis for GENomics. Though admittedly, that one's pretty good and not such a stretch.) I delight in this tradition, and I like to go out of my way to name things in this manner at every opportunity. It's the little choices you make that add whimsy to your life and make things worthwhile. 

## Stages of the project
This project can be broken down into three major stages: 

1. building the pipeline
2. benchmarking the pipeline
3. comparing the pipeline against another established pipeline

The first stage is where the majority of the exploration happens. I read about every step involved in a WES germline short variant discovery pipeline, how they work, and which tools exist to accomplish each step. I then select the tools I want to use for my pipeline and stitch them together. Most of my documentation happens here.

In the second stage, I test my pipeline by benchmarking it with GIAB[^9] truth sets. This will act as a sanity check to verify that my pipeline works the way it's supposed to. 

In the third and final stage, I compare my pipeline's performance to that of an established pipeline. In this case, I'll use the Exome Germline Single Sample Pipeline (v3.1.19)[^10] from the Broad Institute[^11], which is based on the Genome Analysis Toolkit (GATK)[^12], also from the Broad. This stage acts as a way to measure how good my pipeline actually is. I hypothesize that it won't be that good, but the process of working on this project will help me figure out why.

The successful completion of all three stages will, ideally, ensure that when I someday run my own exome through the pipeline, I'll get meaningful results that will make me feel giddy about finally turning myself into the research subject I always knew I was meant to become. 

## Major references
The nature of my project called for a reference that I would use to build the pipeline itself. For this purpose, I have chosen the Exome Germline Single Sample Pipeline (v3.1.19)[^10], developed by the fine folks at the Broad Institute[^11]. I consider it a strong candidate because it is based on GATK, or the Genomic Analysis ToolKit[^12], which is a well-established, widely used set of genomic analysis tools (what it says on the tin, basically) that the developers describe as "the industry standard for identifying SNPs and indels in germline DNA and RNAseq data"[^12]. Indeed, pretty much every study[^13] I've found on the topic of germline variant calling has used GATK and/or identified it as a standard tool. As a result, GATK and its usage are very well documented, making it ideal for someone learning the ropes of genomic/exomic analysis. The Exome Germline Single Sample Pipeline itself is also publicly available in the Broad Institute's GitHub repository[^14], as well as a Terra workspace[^15], meaning I can use study it as a reference *and* use it for myself for the comparison stage of my project.

I mentioned nf-core/sarek[^16] in the README too. This is another good pipeline, but I'll mostly be referencing this for tool comparisons and containerization, both of which will happen down the line. I plan on drawing pretty heavily from sarek for the containerization part, though, which is why I credit it as a major source.

Finally, I *heavily* referenced Genome in a Bottle (GIAB)[^9] from the National Institute of Standards and Technology (NIST) over the course of this project, which is kind of funny because I initially chose to look into it simply because I thought the name was cute. As it turns out, GIAB is a leading source of reference genomes and benchmarking resources, including curated sample and truth datasets and best practices. This means their datasets are publicly available and widely used across studies, especially those that focus on benchmarking. I not only looked to GIAB for my reference genome and benchmark datasets, but I also learned a lot about how to perform and interpret benchmarks from them as well.

## Sources
[^1]: https://en.wikipedia.org/wiki/Germline
[^2]: https://www.cancer.gov/publications/dictionaries/genetics-dictionary/def/germline-variant
[^3]: https://en.wikipedia.org/wiki/Somatic_mutation
[^4]: https://gatk.broadinstitute.org/hc/en-us/articles/360035535932-Germline-short-variant-discovery-SNPs-Indels
[^5]: https://gatk.broadinstitute.org/hc/en-us/articles/9022476791323-Structural-Variants 
[^6]: https://en.wikipedia.org/wiki/Exome
[^7]: https://www.genome.gov/genetics-glossary/Exome
[^8]: https://pmc.ncbi.nlm.nih.gov/articles/PMC2768590/
[^9]: https://www.nist.gov/programs-projects/genome-bottle
[^10]: https://broadinstitute.github.io/warp/docs/Pipelines/Exome_Germline_Single_Sample_Pipeline/README
[^11]: https://www.broadinstitute.org/
[^12]: https://gatk.broadinstitute.org/hc/en-us
[^13]: [You're better off referring to the bibliography for this one. Seriously, every paper cited in there mentions GATK.](bibliography.md)
[^14]: https://github.com/broadinstitute/warp/blob/master/pipelines/wdl/dna_seq/germline/single_sample/exome/ExomeGermlineSingleSample.wdl
[^15]: https://app.terra.bio/#workspaces/warp-pipelines/Exome-Analysis-Pipeline
[^16]: https://nf-co.re/sarek/3.9.0/