# GERMSPIPE: GERMline Short variant discovery PIPEline
A reproducible germline short variant discovery pipeline from Illumina whole-exome sequences, based on the Broad Institute's GATK Best Practices and nf-core/sarek workflows. Personal project.

**This project is currently a work in progress! Please excuse discrepancies and incomplete sections while the project is underway.**

# Description
One day, I decided I wanted to sequence my own genome and run my very own downstream analysis of said genome, because that's an aspiring bioinformatician's idea of a good time. Then I realized it would cost money and take time to get my genome sequenced, even if I just wanted an exome. *Then* I realized that coding is free, so I should learn how to build a variant discovery pipeline while I'm in this current period between not having money and having money. This is the result of that endeavor.

The idea was to read up on existing pipelines, pick them apart, and try to *understand* and *reimplement* the steps involved. Instead of reinventing several wheels, I found existing software and tools for each step, picked the ones I wanted to use, and cobbled them together into my own pipeline. I validated and benchmarked the pipeline, then compared it against an industry standard (the [Exome Germline Single Sample Pipeline v3.1.19 from the Broad Institute](https://broadinstitute.github.io/warp/docs/Pipelines/Exome_Germline_Single_Sample_Pipeline/README)) to measure its performance. 

In other words, my objective for this project is to answer the following questions:

- How does a germline short variant discovery pipeline work?
- Can I build my own version of such a pipeline?
- Does my pipeline work? Importantly, can it run on limited resources (for example, my Lenovo laptop)?
- Does my pipeline hold up against benchmarking? 
- How does my pipeline compare to an established pipeline?

Something that's worth mentioning at this point is that this is a personal project. The point isn't to advance the cutting edge of bioinformatics or whatever; it's for me to learn the fundamentals of the concepts and processes involved in building a bioinformatics pipeline. You're unlikely to find anything here that hasn't already been done better by someone else, but you will find one person's meandering introduction into germline short variant discovery and attempts to figure out what the heck is going on in there.

So anyway, here's how this project is gonna break down. The `src` folder, as you might expect, will contain the pipeline itself. That means the scripts, the dependencies, all that. The `data` folder will contain the reference datasets, the samples, and the truth sets (for benchmarking). The `results` folder will contain all outputs, including intermediary ones. Now, due to the extremely large file sizes typically involved in a bioinformatics project, I did *not* include in this repo any FASTAs, FASTQs, BAMs, VCFs, etc.; instead, I explicitly listed all the files I used to run the pipeline, as well as where I got them, in a [Markdown document](./docs/02_data_acquisition.md). 

The `docs` folder is going to be a little different. Here, I will go into further detail on the structure of my pipeline, talk about each phase, and include the literature I referenced. Note that these documents won't be an instructional overview of a polished product, nor will they be a comprehensive review-style academic paper. Almost everything is exploratory in nature. I guess you could liken this whole thing to a digital lab notebook.

There should also be a Dockerfile containing the exact environment I ran the pipeline in, meaning if you wanted to, you should be able to run it yourself without having to manually hunt down and/or version control dependencies. **As of this writing, I have not started this step, but I'm mentioning it anyway to give you an idea of where I'm trying to be by the end of the project.**

I think it would be cool if someone else got something out of this, but it's okay if not. Either way, thanks for stopping by. 

# Sources and references
These are just the major sources. A full bibliography is provided in `docs/bibliography.md`.
- [Broad Institute GATK Best Practices](https://gatk.broadinstitute.org/hc/en-us/sections/360007226651-Best-Practices-Workflows)
- [Exome Germline Single Sample Pipeline](https://broadinstitute.github.io/warp/docs/Pipelines/Exome_Germline_Single_Sample_Pipeline/README)
- [nf-core/sarek](https://nf-co.re/sarek/3.9.0/)
- [Genome in a Bottle](https://www.nist.gov/programs-projects/genome-bottle)