---
title: "Polishing an assembly"
teaching: 30
exercises: 10
questions:
- "Why do assemblies need to be polished?"
- "What are the different purposes of polishing with short and long reads?"
- "What software can we used to do long and short read polishing?"

objectives:
- "Understand why polishing metagenomes is important."  
- "Understand the different programs used to do short and long read polishing."
- "Use an enviroment in a bioinformatic pipeline."
keypoints:
- "Short reads have a higher base accuracy than long reads and can be used to remove errors in assemblies generated with long reads."
- "Long reads have a lower accuracy but help generate a more contiguous (less fragmented) assembly, so are used to get the structure of the metagenome, but may have small misassemblies or single nucleotide polymorphisms (SNPs)"
- "Medaka is used to polish an assembly with long reads."
- "Pilon is used to polsih an assembly with short reads."
---



In the [previous episode](https://cloud-span.github.io/metagenomics01-qc-assembly/03-assembly/index.html) we generated a draft assembly using Flye. While we could stop there, it is highly likely that our draft assembly contains errors such as gaps and misassemblies. If we generated an assembly using long read (which we just did), the base accuracy of the long reads is also lower, and so without polishing we are still likely to have single nucleotide polymorphisms (SNPs) remaining in the data. We can further improve the quality of this assembly using the raw data we have available in a process known as "assembly polishing".


Long read assemblies have reads that are able to span difficult regions in genomes, that short read assemblies could not. This is helpful particularly for genomes with large repeats. However with long read assemblies the error rate is higher so we introduce SNPs, and we have lower coverage. One way in which we can both reduce the fragmentation of the genome and remove some of these SNPs, is through polishing. We covered this previously in our other course in the episode on [Platform choice](https://cloud-span.github.io/experimental_design01-principles/01-platform/index.html), and in the [introduction section of this course](https://cloud-span.github.io/metagenomics01-qc-assembly/00-introduction-meta/index.html).

## Why bother polishing?

Depending on your downstream use, polishing your assembly may not be essential. However commonly one of the first questions we ask of our metagenome is often, what is in here? In order to answer this question, which we will cover in [Taxonomic annotations](https://cloud-span.github.io/metagenomics03-taxonomic-anno/), you need to compare the sequence in your assembly against a database. If the sequence has SNPs in it and you don't polish your assembly, you may end up with an incorrect annotation. If you are not looking at annotations below the genus level, then the distance to the nearest match may remain the same. However one of the main advantages of using whole genome sequencing rather than amplicon sequencing, is that you can assign annotations to the species level. However not polishing also can have other consequences, if you are using your sequence to match against structural domains to identify the functional of your organism, or you are using it to generate protein predictions, you also may end up with confounding errors downstream. This is discuss in more detail in [Errors in long-read assemblies can critically affect protein prediction](https://www.nature.com/articles/s41587-018-0004-z).

There are two polishing "strategies" that we use. We will be using them in combination. In the further reading of this session you can look at how different combinations of polishing tools can result in an increase in assembly accuracy. We will be performing a long read polishing and following this by polishing with short reads. Typically we would perform the short read polishing step 3 times, each time increasing the base accuracy. To reduce the compute requirements and the time required to finish the assembly, we will just be performing both of these steps once only. Polishing with long reads uses the raw long reads mapped to the assembly to identify misassemblies, typically repeats and contigs that have been incorrectly joined. We will be doing this using a tool called medaka. We will then follow this with a short read polisher, which uses an alignment of the short reads to the assembly to correct SNPs and increase poor quality bases. It will use the higher quality sequqence to substitute errors in our assembly.

## Polishing an assembly with long reads

We're first going to use the filtered raw long reads to polish the draft Flye assembly.
As with the assembly, we need to use polishing software that is especially written for long read raw reads.

[Medaka](https://github.com/nanoporetech/medaka) is a command line tool built by Oxford Nanopore Technologies which will polish an assembly by generating a consensus from raw Nanopore sequences using a recurrent neural network.

We will be using the command medaka_consensus, which is a pipeline that will first align the raw reads to the draft assembly, processes this alignment to generate a pileup which is presented to a recurrent neural network in order to produce a consensus sequence.

Medaka has been pre-installed on the instance so, first we can look at the help page for `medaka_consensus`.
Note: because this is a pipeline made up multiple steps we will be looking at the help documentation for `medaka_consensus` (no space!)

~~~
medaka_consensus -h
~~~
{: .bash}

> ## `medaka_consensus` Help
>
> ~~~
> medaka 1.7.0
>
> Assembly polishing via neural networks. Medaka is optimized
> to work with the Flye assembler.
>
> medaka_consensus [-h] -i <fastx> -d <fasta>
>
>     -h  show this help text.
>     -i  fastx input basecalls (required).
>     -d  fasta input assembly (required).
>     -o  output folder (default: medaka).
>     -g  don't fill gaps in consensus with draft sequence.
>     -r  use gap-filling character instead of draft sequence (default: None)
>     -m  medaka model, (default: r941_min_hac_g507).
>         Choices: r103_fast_g507 r103_hac_g507 r103_min_high_g345 r103_min_high_g360 r103_prom_high_g360 r103_sup_g507 r1041_e82_400bps_fast_g615 r1041_e82_400bps_hac_g615 r1041_e82_400bps_sup_g615 r104_e81_fast_g5015 r104_e81_hac_g5015 r104_e81_sup_g5015 r104_e81_sup_g610 r10_min_high_g303 r10_min_high_g340 r941_e81_fast_g514 r941_e81_hac_g514 r941_e81_sup_g514 r941_min_fast_g303 r941_min_fast_g507 r941_min_hac_g507 r941_min_high_g303 r941_min_high_g330 r941_min_high_g340_rle r941_min_high_g344 r941_min_high_g351 r941_min_high_g360 r941_min_sup_g507 r941_prom_fast_g303 r941_prom_fast_g507 r941_prom_hac_g507 r941_prom_high_g303 r941_prom_high_g330 r941_prom_high_g344 r941_prom_high_g360 r941_prom_high_g4011 r941_prom_sup_g507 r941_sup_plant_g610
>         Alternatively a .tar.gz/.hdf file from 'medaka train'.
>     -f  Force overwrite of outputs (default will reuse existing outputs).
>     -x  Force recreation of alignment index.
>     -t  number of threads with which to create features (default: 1).
>     -b  batchsize, controls memory use (default: 100).
> ~~~
> {: .output}
{: .solution}

* From this we can see that `-i` for the input basecalls (meaning Nanopore raw-reads) and `-d` for the assembly are required.  
* As Medaka uses recurrent neural networks we need to pick an appropriate model (`-m`) for the data we're using. From the [documentation](https://github.com/nanoporetech/medaka#models), Medaka models are named to indicate i) the pore type, ii) the sequencing device (MinION or PromethION), iii) the basecaller variant, and iv) the basecaller version, with the format: `{pore}_{device}_{caller variant}_{caller version}`. Medaka doesn't offer an exact model for our dataset while it is possible to train a model yourself we will not be doing that here and instead will use the closest available model. This is the model `r941_prom_fast_g303`, so we also need to add that to our command as that isn't medakas default.  
* Finally, to speed this step up we need to specify the number of threads with `-t`.

This gives us the command:
~~~
medaka_consensus -i ERR3152367_sub5_filtered.fastq -d assembly.fasta -m r941_prom_fast_g303 -t 4 &> medaka.out &
~~~
{: .bash}
Note, we have added `&> medaka.out &` to redirect the output and run the command in the background. Medaka shouldn't take as long as Flye did in the previous step (probably around 20 mins), but it's a good idea to run things in the background so that you can do other things while the program is running.

Similar to Flye, we can look in the output file (`medaka.out`) to check the progress of the command.
~~~
less medaka.out
~~~
{: .bash}
If the medaka command has been run correctly you will see something like this at the start of the output:
~~~
Checking program versions
This is medaka 1.7.0
Program    Version    Required   Pass     
bcftools   1.15.1     1.11       True     
bgzip      1.15.1     1.11       True     
minimap2   2.24       2.11       True     
samtools   1.15.1     1.11       True     
tabix      1.15.1     1.11       True     
Aligning basecalls to draft
Constructing minimap index.
[M::mm_idx_gen::0.515*0.99] collected minimizers
[M::mm_idx_gen::0.648*1.40] sorted minimizers
[M::main::0.877*1.29] loaded/built the index for 146 target sequence(s)
[M::mm_idx_stat] kmer size: 15; skip: 10; is_hpc: 0; #seq: 146
[M::mm_idx_stat::0.910*1.28] distinct minimizers: 2598367 (94.68% are singletons); average occurrences: 1.076; average spacing: 5.350; total length: 14953273
~~~
{: .output}
Medaka first looks for the other programs that it needs (known as dependencies) and their versions, which have also been pre-installed on the instance. Once it confirms they are preset it begins by aligning the raw reads (basecalls) to the assembly using minimap.
Once medaka has completed the end of the file will contain something like:
~~~
[20:51:16 - DataIndx] Loaded 1/1 (100.00%) sample files.
[20:51:16 - DataIndx] Loaded 1/1 (100.00%) sample files.
[20:51:16 - DataIndx] Loaded 1/1 (100.00%) sample files.
[20:51:16 - DataIndx] Loaded 1/1 (100.00%) sample files.
[20:51:16 - DataIndx] Loaded 1/1 (100.00%) sample files.
[20:51:16 - DataIndx] Loaded 1/1 (100.00%) sample files.
[20:51:16 - DataIndx] Loaded 1/1 (100.00%) sample files.
[20:51:17 - DataIndx] Loaded 1/1 (100.00%) sample files.
[20:51:17 - DataIndx] Loaded 1/1 (100.00%) sample files.
[20:51:17 - DataIndx] Loaded 1/1 (100.00%) sample files.
Polished assembly written to medaka/consensus.fasta, have a nice day.
~~~
{: .output}

Once medaka has completed we can navigate into the output directory and look at the files Medaka has generated.

~~~
cd medaka
ls
~~~
{: .bash}

~~~
calls_to_draft.bam  calls_to_draft.bam.bai  consensus.fasta  consensus.fasta.gaps_in_draft_coords.bed  consensus_probs.hdf
~~~
{: .output}

We can see that medaka has created multiple files, these are the following:

* `calls_to_draft.bam` - this is a BAM file contaning the alignment of the raw reads (basecalls) to the draft assembly
* `calls_to_draft.bam.bai` - this is an index file of the above BAM file
* `consensus.fasta` - this is the consensus sequence, or polished assembly in our case in FASTA format
* `consensus.fasta.gaps_in_draft_coords.bed` - this is a BED file containing information about the location of any gaps in the consensus sequence which can be used when visualising the assembly
* `consensus_probs.hdf` - this is a file that contains the output of the neural network calculations and is not an output for end-users, so we don't need to worry about this file

`consensus.fasta` is the file we're interested in, which in our case is the polished assembly.

> ## BAM and SAM Files
> A [SAM file](https://genome.sph.umich.edu/wiki/SAM), is a tab-delimited text file that contains information for each individual read and its alignment to the genome. While we do not have time to go into detail about the features of the SAM format, the paper by [Heng Li et al.](http://bioinformatics.oxfordjournals.org/content/25/16/2078.full) provides a lot more detail on the specification.  
> The compressed binary version of SAM is called a BAM file. We use this version to reduce size and to allow for indexing, which enables efficient random access of the data contained within the file.  
> The file begins with a header, which is optional. The header is used to describe the source of data, reference sequence, method of alignment, etc., this will change depending on the aligner being used. Following the header is the alignment section. Each line that follows corresponds to alignment information for a single read. Each alignment line has 11 mandatory fields for essential mapping information and a variable number of other fields for aligner specific information. An example entry from a SAM file is displayed below with the different fields highlighted.  
> See [Genomics - Variant Calling](https://cloud-span.github.io/04genomics/01-variant_calling/index.html) for a deeper dive.  
{: .callout}

## Polishing with short reads

We will be using the program [Pilon](https://github.com/broadinstitute/pilon) to further polish the draft assembly using the raw short reads. Pilon will improve a draft assembly by filling gaps, fixing misassemblies and correcting bases. You can read more about how it works in the paper [Pilon: An Integrated Tool for Comprehensive Microbial Variant Detection and Genome Assembly Improvement](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0112963).

Bioinformatics programs are not built equally. Some programs like Flye or Medaka will require very few input files as they will generate any that they need within the pipeline. Some programs however, require a lot of user input to generate the input files that are needed.  

Pilon is in the latter group of bioinformatics software, so we will need to do some pre-processing using other programs to create some of the inputs needed.

### Generating the Pilon input files

We will first use the program [BWA](https://github.com/lh3/bwa) to generate an alignment of the raw short reads against the draft genome.

We've previously covered aligning reads to a genome in [Genomics - Variant Calling](https://cloud-span.github.io/04genomics/01-variant_calling/index.html). We will be using very similar commands here.

We first need to index the polished assembly we got from Medaka. Indexing allows the aligner to quickly find potential alignment sites for query sequences in a genome, which saves time during alignment.
~~~
bwa index consensus.fasta
~~~
{: .bash}
This should only take a few seconds to complete so we don't need to run the job in the background.
Once the indexing is complete you should see an output like:
~~~
[bwa_index] Pack FASTA... 0.51 sec
[bwa_index] Construct BWT for the packed sequence...
[bwa_index] 5.86 seconds elapse.
[bwa_index] Update BWT... 0.10 sec
[bwa_index] Pack forward-only FASTA... 0.10 sec
[bwa_index] Construct SA from BWT and Occ... 1.81 sec
[main] Version: 0.7.17-r1188
[main] CMD: bwa index consensus.fasta
[main] Real time: 8.704 sec; CPU: 8.395 sec
~~~
{: .output}

We can then align the short reads to the draft assembly.

It is possible to chain together commands in unix using a process known as "piping". This allows the output from one command to be directly passed to another command for further processing. This is especially useful for situations where you may not need the intermediate file again. To do this we use the pipe `|` character.  

You can find the pipe (<kbd>|</kbd>) character on your keyboard, usually by typing <kbd>⇧ Shift</kbd> + <kbd>\</kbd>.

You can use multiple pipes in one command but data will only go from the left to the right.
I.e.
`command1 | command2 | command3 | .... |`

We will be using two pipes to join three different steps in order to align the raw reads to the draft assembly and then sort this alignment to generate a sorted BAM file.

First we will generate the alignment using BWA mem, then convert the alignment into BAM with `samtools view` and finally sort the alignment with `samtools sort`. Doing it this way w will only generate the one output file, `short_read_alignment.bam`, avoiding the need to generate large intermediary files we won't need again between the other two steps.

We have run each of these commands separately in [Genomics - Variant Calling](https://cloud-span.github.io/04genomics/01-variant_calling/index.html), if you want to remind yourself of what they do in more detail.

This will also take around 30 minutes so we will use `&> alignment.out &` to redirect the commands process to a file and to run the command in the background.

~~~
bwa mem -t 4 consensus.fasta ERR3152367_sub5_filtered.fastq | samtools view - -Sb | samtools sort - -@4 -o short_read_alignment.bam &> alignment.out &
~~~
{: .bash}
You can check the process of this job by looking at the `alignment.out` file
~~~
less alignment.out
~~~
{: .bash}

~~~
[M::bwa_idx_load_from_disk] read 0 ALT contigs
[M::process] read 8758 sequences (40002050 bp)...
[M::process] read 8872 sequences (40001127 bp)...
[M::mem_process_seqs] Processed 8758 reads in 134.856 CPU sec, 33.785 real sec
[M::process] read 8774 sequences (40005021 bp)...
[M::mem_process_seqs] Processed 8872 reads in 138.347 CPU sec, 34.631 real sec
[M::process] read 8736 sequences (40009331 bp)...
[M::mem_process_seqs] Processed 8774 reads in 130.086 CPU sec, 32.545 real sec
[M::process] read 8848 sequences (40002710 bp)...
[M::mem_process_seqs] Processed 8736 reads in 132.912 CPU sec, 33.277 real sec
[M::process] read 8884 sequences (40009423 bp)...
[M::mem_process_seqs] Processed 8848 reads in 134.169 CPU sec, 33.883 real sec
[M::process] read 8902 sequences (40003755 bp)...
[M::mem_process_seqs] Processed 8884 reads in 129.038 CPU sec, 32.410 real sec
[M::process] read 8760 sequences (40000601 bp)...
~~~
{: .output}
Once completed, the end of the `alignment.out` file should contain something like:
~~~
[M::mem_process_seqs] Processed 8862 reads in 117.475 CPU sec, 29.369 real sec
[M::process] read 4795 sequences (23206538 bp)...
[M::mem_process_seqs] Processed 8610 reads in 125.241 CPU sec, 31.397 real sec
[M::mem_process_seqs] Processed 4795 reads in 87.225 CPU sec, 23.866 real sec
[main] Version: 0.7.17-r1188
[main] CMD: bwa mem -t 4 consensus.fasta ../ERR3152367_sub5_filtered.fastq
[main] Real time: 2454.659 sec; CPU: 9557.109 sec
[bam_sort_core] merging from 4 files and 4 in-memory blocks...
~~~
{: .output}
We should now have genreated the `short_read_alignment.bam` file - this is a binary file (meaning it's not human readable) so we won't be checking it's contents.

We then need to run the following command to index the aligment. We haven't added this command to the above pipe as we need the BAM file from above for further analysis and to be able to index it!
~~~
samtools index short_read_alignment.bam
~~~
{: .bash}
This command will take around a minute so we don't need to run it in the background.

Once your prompt has returned you should also have a file named `short_read_alignment.bam.bai` which is the index.

### Running Pilon
Now we have generated the necessary input files we can, finally, run Pilon.

Pilon has been preinstalled on the instance so we can first view the help documentation with:

~~~
pilon --help
~~~
{: .bash}

> ## Pilon help Documentation
> ~~~
> Pilon version 1.24 Thu Jan 28 13:00:45 2021 -0500
>
>    Usage: pilon --genome genome.fasta [--frags frags.bam] [--jumps jumps.bam] [--unpaired unpaired.bam]
>                 [...other options...]
>           pilon --help for option details
>
>
>         INPUTS:
>           --genome genome.fasta
>              The input genome we are trying to improve, which must be the reference used
>              for the bam alignments.  At least one of --frags or --jumps must also be given.
>           --frags frags.bam
>              A bam file consisting of fragment paired-end alignments, aligned to the --genome
>              argument using bwa or bowtie2.  This argument may be specifed more than once.
>           --jumps jumps.bam
>              A bam file consisting of jump (mate pair) paired-end alignments, aligned to the
>              --genome argument using bwa or bowtie2.  This argument may be specifed more than once.
>           --unpaired unpaired.bam
>              A bam file consisting of unpaired alignments, aligned to the --genome argument
>              using bwa or bowtie2.  This argument may be specifed more than once.
>           --bam any.bam
>              A bam file of unknown type; Pilon will scan it and attempt to classify it as one
>              of the above bam types.
>           --nanopore ont.bam
>              A bam file containing Oxford Nanopore read alignments. Experimental.
>           --pacbio pb.bam
>              A bam file containing Pacific Biosciences read alignments. Experimental.
>         OUTPUTS:
>           --output prefix
>              Prefix for output files
>           --outdir directory
>              Use this directory for all output files.
>           --changes
>              If specified, a file listing changes in the <output>.fasta will be generated.
>           --vcf
>              If specified, a vcf file will be generated
>           --vcfqe
>               If specified, the VCF will contain a QE (quality-weighted evidence) field rather
>               than the default QP (quality-weighted percentage of evidence) field.
>           --tracks
>               This options will cause many track files (*.bed, *.wig) suitable for viewing in
>               a genome browser to be written.
>         CONTROL:
>           --variant
>              Sets up heuristics for variant calling, as opposed to assembly improvement;
>              equivalent to "--vcf --fix all,breaks".
>           --chunksize
>              Input FASTA elements larger than this will be processed in smaller pieces not to
>              exceed this size (default 10000000).
>           --diploid
>              Sample is from diploid organism; will eventually affect calling of heterozygous SNPs
>           --fix fixlist
>              A comma-separated list of categories of issues to try to fix:
>                "snps": try to fix individual base errors;
>                "indels": try to fix small indels;
>                "gaps": try to fill gaps;
>                "local": try to detect and fix local misassemblies;
>                "all": all of the above (default);
>                "bases": shorthand for "snps" and "indels" (for back compatibility);
>                "none": none of the above; new fasta file will not be written.
>              The following are experimental fix types:
>                "amb": fix ambiguous bases in fasta output (to most likely alternative);
>                "breaks": allow local reassembly to open new gaps (with "local");
>                "circles": try to close circlar elements when used with long corrected reads;
>                "novel": assemble novel sequence from unaligned non-jump reads.
>           --dumpreads
>              Dump reads for local re-assemblies.
>           --duplicates
>              Use reads marked as duplicates in the input BAMs (ignored by default).
>           --iupac
>              Output IUPAC ambiguous base codes in the output FASTA file when appropriate.
>           --nonpf
>              Use reads which failed sequencer quality filtering (ignored by default).
>           --targets targetlist
>              Only process the specified target(s).  Targets are comma-separated, and each target
>              is a fasta element name optionally followed by a base range.
>              Example: "scaffold00001,scaffold00002:10000-20000" would result in processing all of
>              scaffold00001 and coordinates 10000-20000 of scaffold00002.
>              If "targetlist" is the name of a file, each line will be treated as a target
>              specification.
>           --verbose
>              More verbose output.
>           --debug
>              Debugging output (implies verbose).
>           --version
>              Print version string and exit.
>         HEURISTICS:
>           --defaultqual qual
>              Assumes bases are of this quality if quals are no present in input BAMs (default 10).
>           --flank nbases
>              Controls how much of the well-aligned reads will be used; this many bases at each
>              end of the good reads will be ignored (default 10).
>           --gapmargin
>              Closed gaps must be within this number of bases of true size to be closed (100000)
>           --K
>              Kmer size used by internal assembler (default 47).
>           --mindepth depth
>              Variants (snps and indels) will only be called if there is coverage of good pairs
>              at this depth or more; if this value is >= 1, it is an absolute depth, if it is a
>              fraction < 1, then minimum depth is computed by multiplying this value by the mean
>              coverage for the region, with a minumum value of 5 (default 0.1: min depth to call
>              is 10% of mean coverage or 5, whichever is greater).
>           --mingap
>              Minimum size for unclosed gaps (default 10)
>           --minmq
>              Minimum alignment mapping quality for a read to count in pileups (default 0)
>           --minqual
>              Minimum base quality to consider for pileups (default 0)
>           --nostrays
>              Skip making a pass through the input BAM files to identify stray pairs, that is,
>              those pairs in which both reads are aligned but not marked valid because they have
>              inconsistent orientation or separation. Identifying stray pairs can help fill gaps
>              and assemble larger insertions, especially of repeat content.  However, doing so
>              sometimes consumes considerable memory.
> ~~~
> {: .output}
{: .solution}

You can read more about the possible outputs Pilon can produce in the [Wiki](https://github.com/broadinstitute/pilon/wiki/Output-File-Descriptions).

We can see there are many different options for pilon, we will be using the defaults for our assembly.
* `--genome` - this will be the output assembly from medaka
* `--unpaired` - the short reads we ysed to create the BAM alignment were unpaired, so we need to specify the unpaired flag
* `--outdir` - we are also going to get pilon to generate a directory for all the output

~~~
pilon --genome consensus.fasta --unpaired short_read_alignment.bam --outdir pilon &> pilon.out &
~~~
{: .bash}

We can again keep track of the analysis by looking at the `pilon.out` file.

When the command is initially run  
~~~
To execute Pilon run: java -Xmx8G -jar $EBROOTPILON/pilon.jar
Adjust the memory value according to the size of your input files
Pilon version 1.23 Mon Nov 26 16:04:05 2018 -0500
Genome: consensus.fasta
Fixing snps, indels, gaps, local
Input genome size: 14973646
Scanning BAMs
short_read_alignment.bam: 932261 reads, 0 filtered, 873025 mapped, 0 proper, 0 stray, Unpaired 100% 3401+/-3002, max 12408
Processing contig_98:1-17822
Processing contig_27:1-7734
Processing contig_156:1-10738
Processing contig_106:1-9517
Processing contig_147:1-98783
Processing contig_96:1-15421
Processing contig_12:1-7084
Processing contig_15:1-70238
Processing contig_129:1-6453
Processing contig_39:1-5805
Processing contig_152:1-14830
Processing contig_33:1-7570
Processing contig_50:1-4058
~~~
{: .output}

When Pilon finishes the end of the file will contain something like:
~~~
Writing updated contig_77_pilon to pilon/pilon.fasta
Writing contig_107:1-4213 changes to pilon/pilon.changes
Writing updated contig_107_pilon to pilon/pilon.fasta
Writing contig_34:1-156470 changes to pilon/pilon.changes
Writing updated contig_34_pilon to pilon/pilon.fasta
Writing contig_38:1-59388 changes to pilon/pilon.changes
Writing updated contig_38_pilon to pilon/pilon.fasta
Writing contig_44:1-22564 changes to pilon/pilon.changes
Writing updated contig_44_pilon to pilon/pilon.fasta
Mean unpaired coverage: 165
Mean total coverage: 165
~~~
{: .output}

We can navigate into the pilon directory and have a look at the output files Pilon has produced.
~~~
cd pilon
ls
~~~
{: .bash}
~~~
pilon.fasta
~~~
{: .output}

We can see pilon has produced a fasta file, which is the newly polished assembly.
This file is now our assembly, in the next episode we will be assessing the quality of this assembly and compare the quality to the previous draft assemblies.

> ## Recommended reading:
> While you're waiting for the polishing to finish here's some things you might want to read about:
> * Comparison of combined assembly and polishing method [Trycycler: consensus long-read assemblies for bacterial genomes](https://link.springer.com/article/10.1186/s13059-021-02483-z)
> * Polishing strategy for ONT and Pacbio Hifi reads [Polishing high-quality genome assemblies](https://www.nature.com/articles/s41592-022-01515-1)
> * Comparison of polishing of ONT data with alignment free tool Jasper compared to POLCA,NextPolish and ntEdit [JASPER: a fast genome polishing tool that improves accuracy and creates population-specific reference genomes](https://www.biorxiv.org/content/10.1101/2022.06.14.496115v1.full)
> * Comparison of short read polishers including pilon to the polisher Polypolish [Polypolish: Short-read polishing of long-read bacterial genome assemblies](https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1009802#)
> * Pilon short read polisher paper [Pilon: An Integrated Tool for Comprehensive Microbial Variant Detection and Genome Assembly Improvement](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0112963)
> * Accuracy of polishers includin medaka for nanopore data [Nanpore consensus quality](https://github.com/rrwick/August-2019-consensus-accuracy-update#racon)
> * Comparison of nanopore polishing tools [Comparative evaluation of Nanopore polishing tools for microbial genome assembly and polishing strategies for downstream analysis](https://www.nature.com/articles/s41598-021-00178-w)

{: .callout}  
