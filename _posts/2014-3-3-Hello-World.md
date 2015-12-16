---
layout: post
title: You're up and running!
---

# Intro
This is the new blog by the Iqbal lab. We will be adding the occasional post to talk about interesting things in the world of genetic variation and pathogens. Watch this space.

# Test driving some assemblers and polishers
Oxford Nanopore MinIONs generate data with a relatively high error rate, for someone accustomed to Illumina data, but these errors are supposed to be sufficiently random for consensus to iron out any issues. Errors are not uniformly distributed either spatially across the read (in fact they are bursty), or in terms of error-type happen as there is some context dependence.

So we wanted to try out a few options.  There are now several pipelines for de-novo assembly of Oxford Nanopore data, including:

 - Nick Loman, Josh Quick and Jared Simpson  (henceforth LQS) created a [pipeline] which uses nanocorrect, Celera Assembler and  2 rounds of nanopolish.  
 - Adam Phillippy's group have a [PBcR pipeline] which takes advantage of the [MinHash Alignment Process] (MHAP) and the Celera Assembler in PBcR, before running 2 rounds of nanopolish. 
 - Heng Li’s [miniasm], which does no correction, but instead tries to assemble straight away.

[pipeline]:<http://www.nature.com/nmeth/journal/v12/n8/full/nmeth.3444.html>
[PBcR pipeline]:<http://wgs-assembler.sourceforge.net/wiki/index.php/PBcR#Assembling_a_MinION_dataset>
[MinHash Alignment Process]:<http://www.nature.com/nbt/journal/v33/n6/full/nbt.3238.html>
[miniasm]:<https://github.com/lh3/miniasm>

All of these pipelines work purely with nanopore data, and we wanted to know how much improvement you could get by combining with Illumina short reads. Since the assembled contigs were likely to differ from the truth by indels, and potentially clustered indels, we did not want to use any method based on pileups. Instead, we thought as a baseline we should see how well de Bruijn graph correction worked, as we have a lot of experience of this. Hence, we came up with Poligraph: a tool for polishing de-novo nanopore assemblies with reads (usually miseq) using de bruijin graphs. It is essentially a wrapper for calling differences between the draft assembly and a set of (high quality) reads with [cortex], and using them to update the assembly.  

[cortex]:<http://cortexassembler.sourceforge.net/>

### How does poligraph work?
The correction process builds a ‘reference’ de Bruijn graph structure of the draft assembly, and a second graph of the high quality reads using cortex. The read graph is cleaned to remove noise. Cortex then creates a vcf of ‘bubble calls’ between the ‘reference’ and the reads, and these calls are used to update the assembly.

### How does it perform?

We’re going to measure assembly in 4 ways:
1.  Number of contigs (in the days of Illumina assemblies we might have talked about N50 etc, but here we simply aim for a single contig assembly)
2.  Structural accuracy – we align using Mummer to a high quality finished assembly for the same sample.
3.  Local per-base accuracy, using dnadiff.
4.  Number of SNP and indel errors remaining


##### Dataset 1: E Coli K-12 MG1655 (SQK0005)
Nick Loman has very helpfully made an E Coli K-12 MG1655 dataset available, along with reads from successive different nanopore chemistries. Multiple different groups have used these data to test their tools, which provides us with data to compare with. We started with the chemistry version SQK005. This is the data that LQS used to produce a single contig assembly in their [paper]. (paper does indeed say this, although I seem to have got 3 contigs)

[paper]:<http://www.nature.com/nmeth/journal/v12/n8/full/nmeth.3444.html>

Tables of results for these assemblies:

|                                    | LQS | PBcR | Miniasm |
|------------------------------------|:-----:|:-----------:|:---------:|
| Aligned Bases                      |99.97%|99.96%|78.85%|
| Percent Identity for Aligned Bases |99.47%|99.24%|84.05|
| Number of Contigs                  |3|1|1|
| Number of SNPs                     |1357|4115|246377|
| Number of Indels                   |22622|30838|365081|
| Time                  |2 weeks|?| 2mins|
The first thing to note is that both the LQS and Phillippy group assembly pipelines performed a correction step before assembly and a polishing step afterwards. For a fair comparison, I should have at least run 2 rounds of nanopolishing on the miniasm assembly as well, but when I tried this on a different sample it took 5 days on 16 cores to run just one round. Various things we see here:

* The LQS assembly has 3 contigs, whereas in their paper (same data but different software version (TRUE?)) they achieved 1 contig
* As expected the (unpolished) miniasm assembly has 100x more SNPs (one every 20bp) and  10x more indels than the other assemblies.
* Two weeks for the LQS assembly does rather ruin the benefit of rapid MinION sequencing, but it does leave us time for an Illumina run ;-)
* We don't know if the low %aligned bases for the miniasm assembly is due to the high error rate, or missing sequence/collapsed repeats (but see below..)
(Mummer plots to show structurally look good.)

These results are astonishing - not an honest-to-god shock as we had read the LQS paper and miniasm tweets - but this is a world away from what we settled for with Illumina assembly. The broad genome structure is correct, though the per-base error rate is higher than would be required for a "finished assembly" (1 error every 100Mb). What does poligraph+Illumina data add to the mix? 
For each of these three draft assemblies, we ran poligraph globally using the same subset of the miseq reads (trimmed and reduced to approximately 100x).

|                                    | LQS | PBcR | Miniasm |
|------------------------------------|:-----:|:-----------:|:---------:|
| Aligned Bases                      |99.97%|100.00%|92.44%|
| Percent Identity for Aligned Bases |99.9569%|99.9567%|96.9092%|
| Number of Contigs                  |3|1|1|
| Number of SNPs                     |112|185|54835|
| Number of Indels                   |1460|1580|79249|
| Time to run poligraph                  |6 mins|7 mins|13 mins|

Various observations:
* The most surprising result is that for the miniasm assembly - in just 15 minutes we have created an assembly which has almost 97% identity
* Poligraph takes the %Aligned Bases up from 79% to 92% for miniasm
* The number of miniasm SNPs drops by a factor of 100 and indels by a factor of 5
* LQS/PBcR each go up to >99.95% accuracy.

Looking also at the mummerplots, we can again see that all three assemblies align nicely to the referece:

![alt text][LQS_ecolik12]
![alt text][PBcR_ecolik12]
![alt text][miniasm_ecolik12]

[LQS_ecolik12]: https://github.com/iqbal-lab/iqbal-lab.github.io/tree/master/images/20151215_poligraph_Ecoli_K12_loman_and_nanopolish.mdelta.ps "LQS"
[PBcR_ecolik12]:https://github.com/iqbal-lab/iqbal-lab.github.io/tree/master/images/20151215_poligraph_Ecoli_K12_PBcR_and_2nd_nanopolish.mdelta.ps "PBcR"
[miniasm_ecolik12]:https://github.com/iqbal-lab/iqbal-lab.github.io/tree/master/images/20151215_poligraph_Ecoli_K12_miniasm.mdelta.ps "Miniasm"


##### Dataset 2: SQK006
Because of the length of time that the correction step took in the LQS pipeline, we did not test it further on other samples.
[we’ve only looked at SNP/indel errors]


So this all looks great so far, but we were keen to test these methods on samples we had studied in detail. We therefore tried the PBcR pipeline and miniasm on 2 clinical samples described below. The correction step in the LQS pipeline was slow enough that we decided not to run it on these.

##### Dataset 3: clinical E.coli sample
This was a clinical isolate from a study of Nicole Stoesser's (http://biorxiv.org/content/early/2015/11/06/030668) where we had a high quality PacBio assembly which had been polished with Illumina data. 
You can find the nanopore reads ?. 

Tables of results and mummer plots

Unlike the K12 Loman dataset, here all the assemblers fail to produce a single-contig assembly. However, all of the assemblies were consistent and agreed structurally with the truth, apart from one putative inversion ?kb long. This could be due to evolution since the origina assembly was performed, we would have to check. 

##### Dataset 4: clinical K.pne sample
BTW we need to specify in the text what the coverage is of nanopore data for all the samples.
The second clinical sample was a K. pneumoniae clinical isolate with 4 plasmids. 

Table of results
Mummer plots
I guess we need to follow Adam's suggestion to check if PBcR can recover the plasmids. I assume this will work:
Again - both assemblers produce assemblies that agree with the truth and each other, and recover all the plasmids. 

If we add poligraph polishing, we get:
Table


### Conclusions
We were struck by the consistency and correctness of the LQS, PBcR and Miniasm assemblies - completely in contrast to the short read assembly worl. Polishing using the electrical event-level information in nanopore data clearly works, though it is currently slow enough to mean the whole process is no faster than an Illumina run. However I expect that could change quite rapidly with performance modifications to the software. For those who need higher accuracy than can currently be achieved with nanopore data, poligraph polishing with illumina data offers accuracies of up to 99.98% extremely rapidly. 

Some things we plan to do:
* Test whether local assembly (using poligraph in windows across the assembly) results in better polishing
* Get the numbers for miniasm+nanopolish

This has not been a full and systematic benchmarking of the many tools available. Most notably we would like to compare its performance and speed to the hybrid assembly method by Schatz and his group (http://biorxiv.org/content/early/2015/01/06/013490) which can achieve accuracies up to 99.99%. Ivan Sović et al have also done an [in depth comparison] of several hybrid and non-hybrid assemblies of this dataset, although they stop at the draft assembly stage. 
[in depth comparison]:<http://www.biorxiv.org/content/biorxiv/early/2015/11/13/030437.full.pdf>

