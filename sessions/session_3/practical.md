---
layout: page
permalink: sessions/session_3/practical
menubar_toc: true
---
<script src="{{ site.baseurl }}/assets/js/vanilla-back-to-top.min.js"></script>
<script>addBackToTop()</script>

<script src="{{ site.baseurl }}/assets/js/copyCodeSnippet.js" defer></script>
<script src="{{ site.baseurl }}/assets/js/copyCodeBlock.js" defer></script>

Before we begin, please login to Biowulf and request an interactive session:

For a reminder on how to log-in to Biowulf, we refer you to this <a href="https://nci-iteb.github.io/tumor_epidemiology_approaches/sessions/session_1/practical#setup-ssh-connection">Biowulf HPC guide</a>. In short:
- (Windows) use PuTTy
- (Mac) enter <code>ssh USERNAME@biowulf.nih.gov</code>{% include code-snippet-copy.html %} to your terminal, then enter your password

Request an interactive session with:
<code>sinteractive</code>{% include code-snippet-copy.html %}

---

The purpose of this practical session is for you to gain hands-on experience performing quality control (QC) on sequencing data. Today’s tasks will include generating and examining FastQC reports of fastq data, followed by adapter trimming. We will then perform some alignment-level quality control on pre-generated alignment files using a variety of tools, and collect these various QC reports into one interactive report file using MultiQC. Finally, we will highlight some QC metrics that are particularly important for targeted sequencing, whole-exome sequencing (WES), or whole-genome sequencing (WGS).

## Running the QC script

We will use two WES samples sequenced with the same exome capture kit (SeqCap EZ Exome + UTR V3) for the bulk of this practical. One sample is derived from frozen tumor tissue (Fresh_Frozen.bam), another is derived from a Formalin-Fixed Paraffin-Embedded (FFPE) tumor (FFPE_Tumor.bam). We will also use a third sample derived from germline frozen tissue in the very beginning of this practical session (LC_LC_IGC-11-1100_A.bam) for a few demonstrations.

Because our tasks for today will require nearly 20 mins computation running time, we will submit our QC script to Biowulf at the very beginning of this section so that it has time to complete.

After logging in, move to your personal data folder to create a folder for the practical session and move to that folder:

{% include code-block-copy.html %}
```
cd /data/$USER/
mkdir practical_session
cd practical_session
```

Copy over the script that will run today’s analyses, like so:

<code>cp /data/classes/DCEG_Somatic_Workshop/Practical_session_3/practical_3_script.sh .</code>{% include code-snippet-copy.html %}

Once that’s copied we’ll go ahead and submit this script to Biowulf and request 8 cpus and 8GB of memory to speed up the analysis:

<code>sbatch --cpus-per-task 8 --mem 8g practical_3_script.sh</code>{% include code-snippet-copy.html %}

If you’ve successfully submitted a job Biowulf should return an eight-digit number corresponding to the ID of your job, like so:

{% include image-modal.html link="practical_assets/1_job_submission_and_ID.png" %}

Once the job is submitted you can check the status of your job by simply typing and entering “sjobs”.

{% include image-modal.html link="practical_assets/sjobs.png" %}

This will give a report of jobs you have queued or running and their status. Alternatively for a graphical display of job status with more advanced statistics you can check the user dashboard on the Biowulf User Dashboard.

You should also see a file slurm-########.out in your directory once your job begins running (check with ls). This is an output file containing the text that would normally be written to the command line if you were to run these programs interactively (i.e. not via Biowulf job submission). It is a log for your script, and if your output is not what you expected or a job fails you would want to check this file to get details for troubleshooting.

Let’s now examine the script using less or vi:

<code>less practical_3_script.sh</code>{% include code-snippet-copy.html %}

<code>vi practical_3_script.sh</code>{% include code-snippet-copy.html %}

All lines beginning with ## are comments that we’ve added to improve readability. These lines are not actual code and are not executed by Biowulf.

{% include image-modal.html link="practical_assets/set_variables.png" %}

The very first line, ‘#!/usr/bin/env bash’, begins with a ‘shebang’ (#!) and tells Biowulf which program to use to run this script, in this case ‘bash’. This line or something similar should be the first line of every bash script you write.

The first line of code sets a variable, 'PATH_TO_PRACTICE3', to store the location of all of our practical session 3 files. We can use this variable in our script, prefixed by a ‘$’, instead of typing the full path “/data/classes/DCEG_Somatic_Workshop/Practical_session_3”.

*We’ve also set a couple of other variables: OUTDIR (where we’ll store the results) and CPUS_PER_COMMAND (how many cpus to use for each command). Regarding the CPUS_PER_COMMAND variable, we need to explicitly tell most bioinformatics programs how many cpus to use as they’re only configured to use one cpu by default. Whenever you have a path or a value like these that are being used more than a couple times in your script, it’s recommended to use variables. It will save you time and effort, reduce the chances of making a typo, and make it easy to reconfigure your script if something changes (e.g. the path of a folder).*

***For more advanced users:*** *We’ll be running some of the QC tools in parallel with each other (i.e. simultaneously) to speed things up. This is accomplished with the ‘&’ and ‘wait’ commands littered throughout our code. This is also why we’re using only 3 cpus per command despite requesting 8 from Biowulf.*

The next section of the script includes a series of ‘module load’ commands:

{% include image-modal.html link="practical_assets/3_load_modules.jpeg" %}

All of these programs are needed to run the following analyses and are not loaded by Biowulf unless we request them. We did not specify versions for these tools, so Biowulf will simply load the default versions.

What follows these sections is the code to run our QC steps. We will examine these commands and reports over the course of this session.

Stop viewing the script by typing ‘:q’ and hitting enter. If you check the output directory now with the command ‘ls practical_3’, you’ll probably see the script has started generating files. While this script continues to run, copy the pre-generated final QC reports so that we can examine the results while our script finishes running:

<code>cp -r /data/classes/DCEG_Somatic_Workshop/Practical_session_3/expected_results ./practical3_expected_results</code>{% include code-snippet-copy.html %}

Once that finishes you should see a directory in your data folder titled “practical3_expected_results” (check with ls).

As previously mentioned, today we will be examining a MultiQC report (learn more <a href="https://multiqc.info" target="_blank">here</a>). MultiQC automatically detects QC reports from 114 QC algorithms for NGS and compiles them into interactive plots. MultiQC can further collect reports for many samples at once, allowing you to easily compare samples side-by-side in a single report.

The MultiQC code is the last step of our script and is quite simple:

{% include image-modal.html link="practical_assets/4_multiqc_command.jpeg" %}
<figcaption class="is-italic is-size-7">
<ul>
<li>--title is an option to title your report</li>
<li>--ignore will ignore results in a directory; in this case we avoided including the output in the 'trimgalore_out' folder</li>
<li>The two references to ‘$OUTDIR’ are the output folder for the report (specified with ‘-o’) and the folder to search for QC reports, respectively</li>
</ul>
</figcaption><br>

To open our MultiQC report you would need to mount your /data/ drive to your local computer (<a href="https://hpc.nih.gov/docs/hpcdrive.html">refresher on how to do this from Biowulf</a>) and then open practical3_expected_results/Practical_3_multiqc_report.html, but to save time we have uploaded the multiqc report here. <a href="https://github.com/NCI-ITEB/tumor_epidemiology_approaches_materials/raw/main/practical_materials/practical_3/Practical_3_multiqc_report.html.gz" target="_blank">Click here to download it</a>. Opening this file should open a window containing the MultiQC report in your internet browser:

{% include image-modal.html link="practical_assets/5_mutiqc_genstat.png" %}

Above is the general statistics section of our MultiQC report. This section provides an overall summary of the most important statistics for our samples. These statistics are aggregated from the other QC reports generated by our script.

There is lots of information already contained within this general statistics section. Already we can see that the frozen tumor sample has similar coverage to the FFPE tumor, but has larger insert sizes, slightly longer read length, and much less sequence duplication. Our germline sample from frozen tissue has similar read length and duplication rate to the frozen tumor.

***For Advanced Users:*** *Just above these statistics, users can configure exactly which statistics to include in the summary table using the ‘Configure Columns’ button, can generate custom plots using the ‘Plots’ function, and can change the appearance of the entire report using the toolbox on the right of the screen. It is certainly worth exploring these functions on your own, but for the sake of time we won’t discuss them here.*

---

## Reviewing FASTQ-based QC results

We’ll begin by examining the FastQC output. If you review the script we submitted you’ll see the command we used to run this:

{% include image-modal.html link="practical_assets/6_fastqc_command.jpeg" %}
<figcaption class="is-italic is-size-7">
<ul>
<li>-t is an option to run fastqc with multiple cpus</li>
<li>the backslashes, “\”, at the end of lines tells bash to ignore the newlines, allowing us to keep writing the fastqc command over several lines. You will see these backslashes in nearly every command we run.</li>
</ul>
</figcaption><br>

Fastqc will generate a graphical quality report for each input sample (which can be in fastq or bam/sam/cram format), which it will title [sample name]_fastqc.html. You can view these reports if you wish, but MultiQC has already incorporated this data for us. The results for FastQC within the MultiQC report are located at the end of the report, so click on the ‘FastQC’ tab in the table of contents on the left to jump there.

---

### Sequence Counts and Duplication

This first FastQC plot gives us an idea of how many sequences are in each sample, and also the level of duplication in each sample.

{% include image-modal.html link="practical_assets/7_fastqc_sequence_counts.png" %}

Hover over each of these bars and you’ll see that the FFPE tumors have more reads, but over 30% of those reads are duplicates.

<!--You can also view this plot as percentages to get a comparison adjusting for total read count:

{% include image-modal.html link="practical_assets/8_fastqc_sequence_counts.jpeg" %}

*You can also see this in the ‘Sequence Duplication Levels’ report from FastQC:*

{% include image-modal.html link="practical_assets/9_fastqc_seqduplev.jpeg" %}

*In this graph every unique sequence is categorized by how much it has been duplicated. We can see that 10.8% of all unique sequences in the FFPE sample were duplicated 10-50 times. The frozen sample has very little duplication, and mostly at very low duplication levels.*-->

***Question: Why does the FFPE sample have a high duplication rate?***

***Answer:*** FFPE samples in general tend to have higher duplication due to DNA degradation and fragmentation. This leads to less diverse sequence libraries, and therefore higher duplication. Samples with moderate duplication and no other serious QC issues can still be useful provided you have adequate sequencing depth and coverage after excluding duplicates. Note that duplication will almost always increase as a function of sequencing depth.

Following the ‘Sequence Counts’ plot we have an important graphic showing the quality of sequences by position within the read.

---

### Sequence Quality and Adapter Content

{% include image-modal.html link="practical_assets/10_fastqc_meanqual.png" %}

By hovering over these lines we can see that all of our samples are at or above quality Phred score 30, and our frozen tissue sample is even above 35.

Quality is expected to gradually decrease towards the end of reads due to limitations of sequencing by synthesis. A very sudden drop in sequencing quality, however, could be indicative of issues with the sequencing run.

The last plot we’d like to highlight is the ‘Adapter Content’ plot. This will show you what percentage of reads contain adapter sequences, per base position. Sequencing facilities will often do one pass of adapter trimming themselves before returning your data, but you may still have adapter content remaining. Here we can see that <5% of our germline frozen tissue reads contain adapter sequences.

{% include image-modal.html link="practical_assets/11_fastqc_adapter_content.png" %}

Our FFPE samples are absent from this graph because they have no adapter content. This is because they’ve already been trimmed. These adapter sequences can only negatively impact our analysis so they should be removed. We will remove these via read trimming.

---

### Read Trimming

Looking back at our submitted script, you will see just below our fastqc command we’ve submitted a job to run Trimgalore. Trimgalore will recognize and remove adapter sequences (either from a common database by default, or specified by the user) and trim low quality 3’ bases. Here is the command we ran in our script:

{% include image-modal.html link="practical_assets/12_trimgalore_command.jpeg" %}
<figcaption class="is-italic is-size-7">
<ul>
<li>the last two lines are our samples to trim.</li>
<li>the ‘--stringency’ option sets the minimum number of base pairs that must overlap with an adapter to count as an adapter sequence.</li>
<li>the ‘--gzip’ option will compress the resulting trimmed fastq file with the gzip algorithm. Our samples in this case are small, but when working with larger files (particularly WGS) it’s critical to compress your fastq files.</li>
<li>the ‘--fastqc’ option asks Trimgalore to rerun FastQC automatically once trimming is complete</li>
</ul>
</figcaption><br>

<a href="https://github.com/FelixKrueger/TrimGalore/blob/master/Docs/Trim_Galore_User_Guide.md">Click to learn more about Trimgalore</a>.

By now our script should have completed trimming, and you should see files trimgalore/trimmed-LC_LC_IGC-11-1100_A_val_1_fastqc.html and trimgalore/trimmed-LC_LC_IGC-11-1100_A_val_2_fastqc.html in the output directory. If you check these new fastqc reports on your local computer (<a href="https://github.com/NCI-ITEB/tumor_epidemiology_approaches_materials/raw/main/practical_materials/practical_3/trimgalore_out.tar.gz" target="_blank">download report here</a>) and scroll to the adapter content graphic, you’ll see our samples no longer have adapter sequences.

{% include image-modal.html link="practical_assets/13_fastqc_adapter_content_2.jpeg" %}

---

## Reviewing alignment-based QC results

After trimming your sequences (if necessary for your samples), the next step in a typical workflow is to align your sequences to a reference genome. Alignment is an hour(s) long, computation-intensive process that may also include additional re-alignment steps, duplicate sequences removal, and more.

We don’t have time for that in this practical, so we will instead focus on the quality control that comes after alignment. For those interested in learning more about alignment, we will instead refer you to [this short tutorial](https://learn.gencore.bio.nyu.edu/alignment/). We will also briefly discuss alignment in session 4.

---

### Alignment Flags with Samtools Flagstat

Just after the trimgalore command we ran the Samtools module ‘flagstat’.  

{% include image-modal.html link="practical_assets/14_samtools_flagstat_command.jpeg" %}
<figcaption class="is-size-7 is-italic">This command has the following simple structure: <code>samtools flagstat [OPTIONS] alignment_input</code>.
<ul>
<li>-@ specifies number of cpus</li>
<li>output is redirected to a .txt file with ‘>’ because the outputs from Samtools simply print to the console by default</li>
</ul>
</figcaption><br>

This tool will collect data on the alignment flags in our data to give us a simple view of how well our sequences mapped.

To find this module in our MultiQC report, click on the Samtools tab in the table of contents on the left.

{% include image-modal.html link="practical_assets/15_samtools_visual.jpeg" %}

We can see the data for each sample by hovering over the points in the graphic (here we’ve changed the color of the fresh frozen tumor sample to red to improve visibility via MultiQC’s highlight function). We’ll choose to focus on the FFPE sample for this module as it has some features worth noting. You can also examine the text version generated directly from samtools by opening the files ‘FFPE_Tumor.flagstat.txt’ and ‘Fresh_Frozen.flagstat.txt’.

{% include image-modal.html classes="center" styles="display: block;margin-left: auto; margin-right: auto; max-width:66%" link="practical_assets/16_fresh_frozen_flagstat.jpeg" %}<br>
{% include image-modal.html classes="center" styles="display: block;margin-left: auto; margin-right: auto; max-width:66%" link="practical_assets/17_FFPE_tumor_flagstat.jpeg" %}<br>

<!--A few of these stats are particularly worth highlighting. Examine the total sequences and the total mapped sequences graphs by hovering over the blue dots. In the case of our FFPE tumor sample for example, 53.14M reads were in our sample and 52.96M mapped.

{% include image-modal.html link="practical_assets/samtools_readsMapped.png" %}

The next tab that’s important to check is the count of properly paired reads. This measures the number of paired reads that are mapped close to one another in the genome, and in an expected orientation (one read on the forward strand, one on the reverse strand). Of our 52.96M mapped reads, 50.5M reads (95.4%) were properly paired.

{% include image-modal.html link="practical_assets/samtools_properlyPaired.png" %}

The remaining 2.4M mapped reads could be improperly paired for a variety of reasons: its paired mate did not map, one of the paired reads mapped incorrectly to a distant part of the genome, or possibly there is a structural variant. In this case the statistics ‘self and mate mapped’ just below shows that only 50.6M pairs of reads were both mapped somewhere in the genome, of which 50.5M were properly paired.

{% include image-modal.html link="practical_assets/18_FFPE_readswithitselfandmate.jpeg" %}

This leaves only 0.1M reads where both reads in the pair mapped but mapped improperly.

Of those 0.1M, ~30,000 pairs of reads had quality mapping to different chromosomes.

{% include image-modal.html link="practical_assets/19_FFPE_mate_mapped_to_dif_chr.jpeg" %}-->

The first three stats give important information: how many reads were in each sample, how many reads passed QC *(this is determined by the sequencer/sequencing facility)*, and how many reads mapped. In our case no reads failed QC, and almost every read in both samples mapped. Hover over the points to get specific numbers at the top of the table

{% include image-modal.html link="practical_assets/samtools_reads_passQC_map.png" %}

Further down this table we have some statistics on how paired reads mapped:

{% include image-modal.html link="practical_assets/samtools_pairedMapping_stats.png" %}

The take away from these stats is how many reads were paired, and how many pairs mapped properly (near one another in the genome and with expected stranded-ness). Most reads were properly paired in both samples which is good. A small number of reads may be improperly paired due to structural variants, but this should be a small fraction of total reads. The vast majority of reads are expected to be properly paired.

One notable stat for our samples is that the FFPE tumor has quite a few supplementary alignments:

{% include image-modal.html link="practical_assets/samtools_suppAlign.png" %}

{% include image-modal.html link="practical_assets/supplementary_dots.png" %}

When a sequencing read could be aligned to two or more distinct regions of the genome with no overlap, this is called chimeric alignment. One of the alignments is arbitrarily designated the “representative alignment” and the others are called “supplementary alignments”. Chimeric alignments can be indicative of structural variations.

However, for FFPE samples, chimeric reads could be artificially generated as part of the sequencing artifacts that can arise from the significant damage to DNA. Therefore, additional care should be taken into account when using FFPE samples for calling structural variants. Additional information about this could be found [here](https://pubmed.ncbi.nlm.nih.gov/30418619/).

---

### Assess Insert Sizes with Picard

Let’s now turn to the output from the Picard module ‘CollectInsertSizeMetrics’. This tool, as the name implies, calculates insert size of our reads. Insert size and read length sound similar, but refer to different concepts. The insert size is the total length of the DNA fragment between the adapters, whereas the read lengths are the number of base pairs being sequenced at either end of the insert.

{% include image-modal.html link="practical_assets/20_insert_size_figure.jpeg" %}
<figcaption class="has-text-centered is-size-7 is-italic"> <a href="https://pubmed.ncbi.nlm.nih.gov/24523726/">F.S. Turner, 2014</a> </figcaption><br>

The command we used to generate this output in our script looks as follows:

{% include image-modal.html link="practical_assets/21_picard.jpeg" %}
<figcaption class="is-size-7 is-italic">
Picard is a java coding language application saved as a java .jar file, which is why the command to run it begins with ‘java -Xmx3g -jar’.
<ul>
<li>‘-Xmx3g’ argument is an instruction to java to use up to 3Gb of memory</li>
<li>-I specifies our bam file input</li>
<li>-O and -H specify the names of the text and histogram outputs, respectively</li>
</ul>
</figcaption><br>

Let’s examine this report in MultiQC by clicking on ‘Picard’ in the left table of contents. Hover your mouse over the lines to see the samples they correspond to and the number of sequences at each insert size.

{% include image-modal.html link="practical_assets/22_insert_size_vis.png" %}

One difference that’s very obvious is that the FFPE samples all have relatively small insert sizes, peaking at 115bp. This is almost identical to our sequence length of 112bp, and means that most paired reads in this sample totally overlap. The effective sequencing depth for this FFPE tumor is therefore roughly half of the estimated overall depth as both reads in most pairs contain the same information.

By contrast, the mean insert size of the frozen sample is 325bp and includes many inserts between 500-700bp. Do note that insert size can be too large; if your inserts are many times larger than your sequencing length this could reduce evenness of coverage.

***Question: Why are these distributions so different, and what might be the downstream consequences?***

***Answer:*** A good distribution of insert size for short read WES and WGS data is a strong peak around 300-500bp. Due to unavoidable DNA degradation in FFPE samples, this sample peaks at insert size ~115bp. Small insert sizes will cause higher adapter content in your reads, limit read length, increase duplication rate, make it difficult to discover and resolve structural variants or rearrangements, and possibly impose other discovery limitations depending on your study design.

*Note: you may recall that although we just said FFPE samples often have more adapter content, we didn’t observe any adapters in this FFPE sample with FastQC. This is because the fastq samples for this FFPE tumor were already adapter trimmed.*

---

### Calculate Genome Coverage with Mosdepth

The last QC metric we will discuss is sequencing coverage over the genome. In addition to learning how evenly your sequencing covers the genome, you can also check the sex of your sample using per-chromosome coverage statistics. This will allow you to verify the integrity of your sample by comparing the observed sex to the expected sex of the sample.

To do this we ran the tool mosdepth in our script:

{% include image-modal.html link="practical_assets/23_mosdepth_command.jpeg" %}
<figcaption class="is-italic is-size-7">
<ul>
<li>-t is the number of cpus to use</li>
<li>-b is a critical parameter to note; this is an option to restrict coverage calculation to certain parts of the genome, as specified with a BED file</li>
<li>finally we specify the name for the report followed by the input sample in the last line, respectively</li>
</ul>
</figcaption><br>

The -b option in the mosdepth command is very important to note. Recall that the samples we analyzed were whole exome sequencing samples, thus it would be misleading to include non-coding regions of the genome in our calculation. For that reason we’ve specified a BED file with ‘-b’ to restrict the coverage calculation to only the regions of the exome which we sequenced. The BED file you use must be specific to the kit you use for exome capture prior to sequencing.

The third line of each command is the naming convention for our outputs (prefixing them with ‘$OUTDIR/mosdepth/’ will ensure they are deposited in the right folder) followed by the sample to analyze. Now let’s examine the report in MultiQC by clicking on mosdepth in the table of contents.

{% include image-modal.html link="practical_assets/24_mosdepth_vis.png" %}

From the general statistics table at the top of the report, we can see that the median coverage is ~21X for the frozen tumor and ~20X for the FFPE tumor. Now let’s look at the coverage per chromosome:

{% include image-modal.html link="practical_assets/25_mosdepth_cov_per_chr.png" %}

Here we can see that coverage fluctuates between chromosomes, but is consistently between 20-30x. Note the sex chromosomes on the right: our frozen tumor has virtually no coverage on the Y chromosome, which is consistent with the expected sex of this sample (female). The FFPE sample, however, is more ambiguous with ~17.5X coverage of the X chromosome and ~6X coverage of the Y chromosome.

***Question: Records indicate this FFPE sample was derived from a female sample, so why does this sample have coverage of the Y chromosome?***

***Answer:*** There are two possible reasons. One is that the FFPE sample has been cross-contaminated or swapped with a sample from a male individual. Additional QC can verify this by estimating the intersample contamination rate (e.g., VerifyBamID). The other possibility is incorrect mapping to chromosome Y, particularly from low quality reads. This is especially likely on chromosome Y due to its complex features (lower mappability, many repeat regions, high proportion of heterochromatin structure, etc.). This can occur even in uncontaminated female samples. A more strict approach is necessary in these cases.

---

### Assess Sex Concordance with Somalier

At the end of our script we ran the tool <a href="https://github.com/brentp/somalier/releases">Somalier</a>. Somalier calculates a relatedness score between samples using a list of common SNPs, which can be useful for identifying related individuals and sample swaps. Additionally, Somalier calculates a few useful statistics for sex concordance, namely zygosity of chromosome X SNPs.

The Somalier commands are at the end of this script and require running ‘somalier extract’ on each sample to extract SNP information, followed by ‘somalier relate’ to calculate relatedness and other statistics between samples.

{% include image-modal.html link="practical_assets/26_somalier_commands.jpeg" %}
<figcaption class="is-italic is-size-7">
somalier extract
<ul>
<li>-d is the output directory for the results</li>
<li>-s is a list of SNPs in the genome in VCF format which Somalier will sample for calculating relatedness and sex statistics</li>
<li>-f is the specific reference genome used for sequence alignment</li>
<li>the last line is the sample input</li>
</ul>
somalier relate
<ul>
<li>-o is the output directory for the results</li>
<li>all somalier extraction files must then be listed to relate together</li>
</ul>
</figcaption><br>

Somalier has been included in our MultiQC report. Click on Somalier in the table of contents, but note that we won’t look at the automatically generated plots (these plots mostly show relatedness which is useful for larger studies but not relevant for this use-case). Instead, just above the Somalier summary statistics table click on the ‘Plot’ option:

{% include image-modal.html link="practical_assets/27_somalier_stats.png" %}

In the window that appears, select ‘HomAltVarX’ and ‘HetVarX like so.’

{% include image-modal.html classes="center" styles="display: block;margin-left: auto; margin-right: auto; max-width:66%" link="practical_assets/28_somalier_plot.png" %}

Here we can see that our FFPE tumor has very few heterozygous SNPs on the X chromosome, which is unusual for a female with two X chromosomes. Compare this to the fresh frozen tumor where the ratio of heterozygous and homozygous SNPs is roughly 50/50. In combination with relatively high chromosome Y coverage in the FFPE tumor, this is strong evidence for a contamination issue. (NOTE: Somalier will also assess X and Y coverage, but it calculates this coverage at some SNP locations; because many of these SNPs are not within the exome, the coverage calculation will be inaccurate for WES).

Indeed, examine the image below captured from IGV (Integrated Genomics Viewer); visualizing the alignment for this sample shows consistent, exome-specific coverage on the Y chromosome in the FFPE tumor sample, but no coverage in the frozen sample (the histograms in the middle of the image). This indicates that false-positive mapping is an unlikely explanation for the observed Y chromosome coverage.

{% include image-modal.html link="practical_assets/29_IGV_ffpe_fresh_frozen.jpeg" %}

---

## Summary of the Effect of FFPE on NGS Quality

As noted throughout this practical FFPE samples often have several quality issues, such as a high proportion of PCR duplicate reads and very short insert sizes. Consequently the overall coverage, without duplicates and considering the overlap between two paired reads, should always be calculated to determine if you have enough depth. A high proportion of PCR duplicates can be an indication that your starting DNA was highly fragmented, and you may also observe a higher number of artifacts during variant calling. Samples with an extremely high PCR duplicate rate may have to be excluded if the depth of coverage (without PCR duplicates) is too low. A high proportion of unmapped reads is also a red flag, as well as a high proportion of mates mapped to a different chromosome.

Additional QC at the variant level and a strict bioinformatic post-processing of the variant calling are both strongly recommended when analyzing FFPE samples. We will cover this in Session 4: Mutation Calling and Analysis.

---

## Sequencing Strategy-specific QC Metrics

The previous QC metrics we’ve introduced are useful for almost any NGS application, regardless of study design and sequencing methodology. Now we will focus on QC metrics that are particularly suited or only applicable to certain experimental designs.

---

### Targeted Sequencing and Whole Exome Sequencing

The intent of targeted and exome sequencing is to focus your sequencing efforts on a relatively small area of the genome expected to have the greatest importance to your disease of interest. Thus it’s useful to know how well you sequenced your target region(s), and how much of your sequencing was off target. This is something we can answer using another function of Picard, ‘CollectHsMetrics’ (Collect Hybrid Selection Metrics). Here’s an example for how to do this:

{% include image-modal.html link="practical_assets/collectHS_command.png" %}
<figcaption class="is-italic is-size-7">
<ul>
<li>-I is the input alignment file</li>
<li>-O is the output file name</li>
<li>-R is the specific reference genome used for alignment</li>
<li>BAIT_INTERVALS and TARGET_INTERVALS are the regions of the genome targeted during sequencing; they are only minorly different and the target intervals can be used for both</li>
</ul>
</figcaption><br>

This code is included at the end of our script in comment form, and the outputs can be found in the Picard-CollectHsMetrics/ folder of our practical 3 resources folder (/data/classes/DCEG_Somatic_Workshop/Practical_session_3/).

The BAIT_INTERVALS and TARGET_INTERVALS are two subtly different genome interval files; the targets are the exact intervals corresponding to the areas of the genome you wanted to sequence (e.g. gene TP53) and the baits include those targets, plus any extra bases in the DNA baits used for exome target/capture (e.g. small overhangs up and downstream of a target). CollectHsMetrics requires both to be specified, but for most cases it’s perfectly fine to use a single target file for both.

Let’s examine the output with ‘less Fresh_Frozen_output_hs_metrics.txt’:

{% include image-modal.html link="practical_assets/31_fresh_frozen_hs_metrics.jpeg" %}

Since this returns a rather lengthy and hard-to-read report, we will extract the pieces of information we need with ‘cut’, which extracts columns of data that we request by column number. First quit less with :q, then enter the following:

<code>grep -A 2 "## METRICS" Fresh_Frozen_output_hs_metrics.txt | tail -2 | cut -f 46,44,48,49,50</code>{% include code-snippet-copy.html %}

{% include image-modal.html link="practical_assets/32_fresh_frozen_hs_metrics_2.jpeg" %}

The stats we’re looking at in order are the percentage of off-target sequences expressed in units of base pairs, followed by the percent of target bases covered at least 1x, 10x, 20x, and 30x. In this fresh frozen tumor, ~32% of sequenced bases fell off target, however, we do still have at least 1x coverage over 98% of our target and 10x coverage over 82% of the target. If we examine the FFPE tumor, we’ll see it has less off-target sequencing but slightly less even target coverage as well.

{% include image-modal.html link="practical_assets/33_ffpe_hs_metrics.jpeg" %}

---

#### IGV

IGV is an extremely useful resource for visualizing the alignment in areas of any genome and is very commonly used for manually verifying the results of bioinformatics algorithms.

Since we will use IGV locally on your computer (i.e. not on Biowulf), you’ll need the alignment files on your computer. Rather than download from Biowulf, please <a href="https://github.com/NCI-ITEB/tumor_epidemiology_approaches_materials/raw/main/practical_materials/practical_3/IGV_GSTM_alignments.tar.gz" target="_blank">download the bam files we’ve prepared here</a>, which have been isolated to only the region around GSTM1 and GSTM2 using ‘samtools view’. Once that’s done, open IGV on your local computer and follow these steps:

- Ensure that the genome selector in the top left is set to ‘Human (GRCh37/hg19)’. If not, click that selector, click on ‘More...’, and then select ‘Human (hg19)’ and hit ‘OK’.
- Click File in the top left and click “Load from File”.
- Locate and open the files Fresh_Frozen_GSTM.bam, FFPE_Tumor_GSTM.bam, and LC-LC-IGC-11-1100_A_GSTM.bam which we’ve downloaded, as well as the exome annotations BED file for our exome capture kit

{% include image-modal.html link="practical_assets/IGV_setup.png" %}

This should now open our samples in the viewer, though you won’t actually see any data until you pick somewhere in the genome to zoom in. Let’s zoom in on GSTM1 to see what alignments look like. To do this, enter ‘GSTM1’ into the box at the top.

{% include image-modal.html link="practical_assets/GSTM_search.png" %}

This will automatically search for genes matching this pattern. Click GSTM1 to automatically jump to GSTM1 in the genome browser.

{% include image-modal.html link="practical_assets/GSTM_alignment.png" %}

Look at the bottom of the screen, specifically the blue rectangles. The first track of blue rectangles corresponds to the Refseq Genes annotation, and importantly the second track of blue rectangles are the regions of the genome being sequenced by our exome kit. In the middle of the screen, the gray rectangles correspond to our sequencing reads and the histogram above those reads is a representation of the depth of sequencing per base position. From this graph you can see that our sequencing data lines up well with the capture regions of our exome kit. You can also see that the third sample has a distinct lack of coverage over GSTM1. This is a strong indicator of a deletion in this sample.

When you encounter unexpected or surprising results, or you just want to verify the results from an algorithm, using IGV to visually examine the alignment is often one of the first steps taken as a sanity check. You may find that the region of the genome in question is highly repetitive, has some other unique feature affecting your algorithm, or the reads mapping to the region do not actually map very well.

---

### Whole Genome Sequencing

Based on our Sherlock-Lung study, we recommend several additional steps for quality control of whole-genome sequencing data. We will discuss some of these analyses in future sessions.

- We suggest removing any tumor samples with median sequencing depth less than 40x and removing any normal samples less than 25x. These thresholds will allow sufficient read depth for detecting somatic variants with enough germline reads as a control.
- Cross-sample contamination can be identified using genotype concordance. This can be done with Conpair or verifyBamID. We recommend removing any sample with a contamination level above 5%.
- For non-family studies, we keep only one subject with high tumor sequencing depth if two samples from different subjects have relatedness >0.25. The relatedness among all subjects can be estimated with Somalier. We also use Somalier to screen for labeling issues among tumor-normal pairs (e.g., mislabeled IDs).
- If many large copy number events are observed in normal samples, as detected by SCNA algorithms (e.g., Battenberg), this indicates tumor and normal samples may be swapped or mislabeled during sequencing. Paired tumor-normal samples may be reanalyzed by switching tumor and normal status.
- Mutational signatures can be used to infer cancer types or treatments. For example, in our Sherlock-Lung study, we observed some tumor samples with  mutational signatures associated with ultraviolet light and platinum-based drug chemotherapy. These are probable melanoma metastases  and need to be removed for downstream analysis.
- For analysis of somatic alterations, tumors with low purity (e.g., <0.2) and few mutations (e.g. <100) should be removed. Additionally, for tumor evolution analysis it is important to estimate the number of reads per tumor chromosomal copy (NRPCC, see the reference here). We recommend including only tumor samples with NRPCC>10 for tumor evolution analysis.
