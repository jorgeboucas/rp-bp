
# Running the Rp-Bp and Rp-chi pipelines

The Rp-Bp and Rp-chi pipeline consist of an index creation step, which must be performed once for each genome and set of annotations, and a two-phase prediction pipeline, which must be performed for each sample. 

In the first phase of the prediction pipeline, a filtered genome profile is created. In the second phase, the ORFs which show evidence of translation in the profile are predicted.

This document describes the steps to run each of these phases. It shows some sample calls. For all programs, the `--help` option can be given to see the complete list of parameters.

<a id="toc"></a>

* [Creating reference genome indices](#creating-reference-genome-indices)
* [Running Rp-Bp, Rp-chi pipelines](#running-pipelines)
    * [Creating filtered genome profiles](#creating-filtered-genome-profiles)
    * [Predicting translated open reading frames](#predicting-translated-open-reading-frames)
* [Logging options](#logging-options)
* [Parallel processing options](#parallel-processing-options) (SLURM options)

<a id="creating-reference-genome-indices"></a>

## Creating reference genome indices

This section describes the steps necessary to prepare a reference genome and matching annotation for use in the Rp-Bp and Rp-chi pipelines. The process must only be run once for each reference genome and set of annotations.

The entire index creation process can be run automatically using the `prepare-genome` scripts. It reads most of the required paths from a [YAML](http://www.yaml.org/start.html) configuration file. Additionally, it automatically creates some of the output paths.

The script accepts a `--overwrite` flag. Unless this is given, then steps for which the output file already exists will be skipped.

It also accepts a `--do-not-call` flag. If this flag is given, then the commands below will be printed but not executed.

[Logging options](#logging-options) can be given to this script.

[Parallel processing options](#parallel-processing-options) can be given to this script.

### Configuration file keys

The following keys are read from the configuration file. Keys with [`brackets`] are optional.
* `gtf`. The path to the reference annotations
* `fasta`. The path to the reference genome sequence
* `ribosomal_fasta`. The path to the ribosomal sequence


* `genome_base_path`. The path to the output directory for the transcript fasta and ORFs
* `genome_name`. A descriptive name to use for the created files
* `ribosomal_index`. The base output path for the Bowtie2 index of the ribosomal sequence


* [`orf_note`]. An additional description used in the filename of the created ORFs
* [`start_codons`]. A list of strings that will be treated as start codons when searching for ORFs. default: [`ATG`]
* [`stop_codons`]. A list of strings that will be treated as stop codons when searching for ORFS. default: [`TAA`, `TGA`, `TAG`]
* [`sjdb_overhang`]. The value to use for splice junction overlaps when constructing the STAR index. default: 50

* [`ignore_parsing_errors`]. If this key is in the config file with any value, then transcript headers which do not parse correctly will be skipped. A warning is printed for each header skipped in this manner. Otherwise, the program will report the parsing error and quit. default: False

* [`novel_id_re`]. As a part of ORF extraction, the ORFs are assigned a label based on their position relative to annotated CDS regions; part of the labeling process considers the transcript identifiers. This option is a regular expression for the identifiers of novel ORFs (i.e., those from transcripts assembled using StringTie or Cufflinks. ORFs are annotated as novel if their id's match this expression and they would otherwise be annotated as 'noncoding' or 'suspect_overlap'. If no expression is given, no ORFs are annotated as novel.

### Input files

The required input files are those suggested by the configuration file keys.

* `gtf`. The GTF/GFF3 file containing the reference annotations. This file must be compatible with [gffread](http://cole-trapnell-lab.github.io/cufflinks/file_formats/#the-gffread-utility) for extracting coding sequences. Typically, this means at least the `exon` features must be present, and the transcript identifiers must match for exons from the same transcript. Furthermore, the ORFs are labeled based on their positions relative to annotated coding sequences. This labeling is based on the `CDS` information output by `gffread`. Thus, for it to work correctly, the `CDS` features must also be present in the annotation file.


* `fasta`. The input fasta file should contain all chromosome sequences. The identifiers must match those in the GTF file. Typically, the "dna.toplevel.fa" file from Ensembl contains the appropriate sequences and identifiers.


* `ribosomal_fasta`. The ribosomal DNA sequence is typically repeated many times throughout the genome. Consequently, it can be difficult to include in the genome assembly and is often omitted. Therefore, a separate fasta file is required for these sequences (which are later used to filter reads). 

### Output files

* `genome_base_path`/transcript-index/`genome_name`.transcripts.fa
* `genome_base_path`/transcript-index/`genome_name`.genomic-orfs.`orf_note`.bed.gz
* `ribosomal_index`. The bowtie2 index files (`ribosomal_index`.1.bt2, etc.) for the ribosomal_fasta file
* `genome_base_path`/STAR/`genome_name`/. The STAR index files (`SA`, `Genome`, etc.) for the `fasta` and annotated transcripts in the `gtf` file


```python
prepare-genome WBcel235.79.chrI.yaml --num-cpus 10 --mem 100G
```

[Back to top](#toc)

<a id='running-pipelines'></a>

## Running Rp-Bp, Rp-chi pipelines

The entire Rp-Bp pipeline can be run on a set of riboseq samples which all use the same genome indices using a sample sheet-like [YAML](http://www.yaml.org/start.html) configuration file with the `process-all-samples` program. The configuration file respects all of the optional configuration options specified below.

The script accepts a `--tmp <loc>` flag. If this flag is given, then all relevant calls will use `<loc>` as the base temporary directory. Otherwise, the program defaults will be used.

The script accepts a `--overwrite` flag. Unless this is given, then steps for which the output file already exists will be skipped.

[Logging options](#logging-options) can be given to this script.

[Parallel processing options](#parallel-processing-options) can be given to this script.

### Configuration file keys

The following keys are read from the configuration file. Their semantics is exactly as described below. (This text is in both places for convenience.)

* `riboseq_data`. The base output location for all created files.
* [`note`]. An optional string which will be added to all filenames. It should not contain spaces or any other special characters.
* `models_base`. The base path to the compiled models.

**N.B.** The models specified in the paper are included with the source distribution and compiled/pickled as part of the installation process. They are in the folders `/path/to/rp-bp/models/periodic` and `/path/to/rp-bp/models/nonperiodic`. For a standard installation, the `models_base` should be `/path/to/rp-bp/models`.

#### Reference genome options
These options should be exactly the same as those used in the configuration file used to create the reference indices.

* `gtf`. The path to the reference annotations
* `genome_base_path`. The path to the output directory for the transcript fasta and ORFs
* `genome_name`. A descriptive name to use for the created files
* `ribosomal_index`. The base output path for the Bowtie2 index of the ribosomal sequence
* `fasta`. The path to the reference genome sequence

#### Samples specification
* `riboseq_samples`. A dictionary in which each entry specifies a sample. The key is an informative name about the sample, and the value gives the complete path to the sequencing file (a fastq or fastq.gz file). The names will be used to construct filenames, so they should not contain spaces or other special characters.


```python
process-all-samples c-elegans-test.yaml --tmp /home/bmalone/tmp/ --num-cpus 10 --logging-level INFO
```

[Back to top](#toc)

<a id='creating-filtered-genome-profiles'></a>

## Creating ORF profiles

The entire profile creation process can be run automatically using the `create-orf-profiles` script. It reads most of the required paths from a [YAML](http://www.yaml.org/start.html) configuration file. Additionally, it automatically creates some of the output paths.

The script accepts a `--overwrite` flag. Unless this is given, then steps for which the output file already exists will be skipped.

It also accepts a `--do-not-call` flag. If this flag is given, then the commands below will be printed but not executed.

[Logging options](#logging-options) can also be given to this script.

[Parallel processing options](#parallel-processing-options) can be given to this script.

### Configuration file keys

The following keys are read from the configuration file. Keys with [`brackets`] are optional.

* `riboseq_data`. The base output location for all created files.
* [`note`]. An optional string which will be added to all filenames. It should not contain spaces or any other special characters.
* `models_base`. The base path to the compiled models.

**N.B.** The models specified in the paper are included with the source distribution and compiled/pickled as part of the installation process. They are in the folders `/path/to/rp-bp/models/periodic` and `/path/to/rp-bp/models/nonperiodic`. For a standard installation, the `models_base` should be `/path/to/rp-bp/models`.

#### Reference genome options
These options should be exactly the same as those used in the configuration file used to create the reference indices.

* `gtf`. The path to the reference annotations
* `genome_base_path`. The path to the output directory for the transcript fasta and ORFs
* `genome_name`. A descriptive name to use for the created files
* `ribosomal_index`. The base output path for the Bowtie2 index of the ribosomal sequence


#### Flexbar options
* [`adapter_file`]. A fasta file containing a set of adapter sequences used by flexbar. default: None
* [`max_uncalled`]. The maximum number of Ns to permit in a read without filtering. default 1
* [`pre_trim_left`]. The number of bases to remove from the 5' end of all reads: default: 0
* [`adapter_sequence`]. A single sequence used to remove adapters within flexbar. default: None
* [`quality_format`]. The quality format of the reads in the raw fastq file. default: "sanger"

#### STAR options
* [`align_intron_min`]. default: 20
* [`align_intron_max`]. default: 100000
* [`out_filter_intron_motifs`]. default: RemoveNoncanonicalUnannotated
* [`out_filter_mismatch_n_max`]. default: 1
* [`out_filter_mismatch_n_over_l_max`]. default: 0.04
* [`out_filter_type`]. default: BySJout
* [`out_sam_attributes`]. default: AS NH HI nM MD

#### Metagene periodicity options
* [`seqids_to_keep`]. If this list is given, then only transcripts appearing on these identifiers will be used to construct the metagene profiles (and other downstream analysis). The identifiers must match exactly (e.g., "2" and "chr2" do not match)


* [`metagene_profile_start_upstream`]. The number of bases upstream of the translation initiation site to begin constructing the metagene profile. default: 50

* [`metagene_profile_start_downstream`]. The number of bases downstream of the translation initiation site to end the metagene profile. default: 20

* [`metagene_profile_end_upstream`]. The number of bases upstream of the translation termination site to begin constructing the metagene profile. default: 50

* [`metagene_profile_end_downstream`]. The number of bases downstream of the translation termination site to end the metagene profile. default: 20


* [`periodic_offset_start`]. The position, relative to the translation initiation site, to begin calculating periodicity Bayes factors (inclusive)
* [`periodic_offset_end`]. The position, relative to the translation initiation site, to stop calculating periodicity Bayes factors (inclusive)
* [`metagene_profile_length`]. The length of the profile to use in the models. `metagene_profile_length` + `periodic_offset_end` must be consistent with the length of the extracted metagene profile

* [`metagene_profile_iterations`]. The number of iterations to use for each chain in the MCMC sampling. The first half of the iterations are discarded as burn-in samples. All of the remaining samples are used to estimate the posterior distributions. That is, we do not use thinning. default: 500


#### Periodicity and offset options
* [`use_fixed_lengths`]. If this variable is present in the config file with any value, then the estimated  periodic read lengths and offsets will not be used. Instead, fixed values given by `lengths` and `offsets` (below) will be used.

* [`lengths`]. A list of read lengths which will be used for creating the profiles if the `use_fixed_lengths` option is given. Presumably, these are lengths that have periodic metagene profiles.

* [`offsets`]. The P-site offset to use for each read length specifed by `--lengths` if the `use_fixed_lengths` option is given. The number of offsets must match the number of lengths, and they are assumed to match. For example `lengths` of 26, 29 with `offsets` 9, 12 means only reads of lengths 26 bp and 29 bp will be used to create the profiles. The 26 bp reads will be shifted by 9 bp in the 5' direction, while reads of length 29 bp will be shifted by 12 bp.

* [`min_metagene_profile_count`]. If fixed lengths are not used: the minimum number of reads for a particular length in the filtered genome profile. Read lengths with fewer than this number of reads will not be used. default: 1000

* [`min_metagene_bf_mean`]. If fixed lengths are not used: if max_metagene_profile_bayes_factor_var is not None, then this is taken as a hard threshold on the estimated Bayes factor mean. 

 If min_metagene_profile_bayes_factor_likelihood is given, then this is taken as the boundary value; that is, a profile is "periodic" if:

       [P(bf > min_metagene_bf_mean)] > min_metagene_bf_likelihood

 If both max_metagene_bf_var and min_metagene_bf_likelihood are None, then this is taken as a hard threshold on the mean for selecting periodic read lengths.

 If both max_metagene_bf_var and min_metagene_bf_likelihood are given, then both filters will be applied and the result will be the intersection. 
 
 default: 5

* [`max_metagene_bf_var`]. If fixed lengths are not used: if given, then this is taken as a hard threshold on the estimated Bayes factor variance. default: None, i.e., this filter is not used. (null in yaml)

* [`min_metagene_bf_likelihood`]: If fixed lengths are not used: if given, then this is taken a threshold on the likelihood of periodicity (see min_metagene_bf_mean description for more details). default: 0.5

#### Smoothing options

* [`smoothing_fraction`]. The fraction of the profile to use for smoothing within LOWESS. default: 0.2

* [`smoothing_reweighting_iterations`]. The number of reweighting iterations to use within LOWESS. Please see the statsmodels documentation for a detailed description of this parameter. default: 0

* [`min_orf_length`]. If this value is greater than 0, then ORFs whose length (in nucleotides) is less than this value will not be smoothed, and neither the Bayes factor estimates nor the chi-square p-value will be calcualted. default: 0

* [`max_orf_length`]. If this value is greater than 0, then ORFs whose length (in nucleotides) is greater than this value will not be smoothed, and neither the Bayes factor estimates nor the chi-square p-value will be calcualted. default: 0

* [`min_signal`]. ORFs for which the number of in-frame reads is less than this value will not be smoothed, and neither the Bayes factor estimates nor the chi-square p-value will be calculated. default: 5


#### Shared MCMC options
These affect the MCMC both for estimating metagene profile periodicity and ORF translation Bayes factors.
* [`seed`]. The random seed for the MCMC sampling. default: 8675309
* [`chains`]. The number of chains to use in the MCMC sampling. default: 2



### Input files

The required input files are only those suggested by the configuration file keys.

* `ribosomal_index`.
* `gtf`.
* The STAR index
* The periodic models, taken as all `.pkl` files located in `models_base/periodic`.
* The nonperiodic models, taken as all `.pkl` files located in `models_base/nonperiodic`.

### Output files

This script primarily creates the following files. (STAR also creates some temporary and log files.)

* Trimmed and quality filtered reads
    * **trimmed and filtered reads**. A fastq.gz file containing the reads after removing adapters and low-quality reads. `riboseq_data`/without-adapters/`sample-name`[.`note`].fastq.gz
    
* Reads aligning to ribosomal sequences
    * **discarded reads**. A fastq.gz file containing reads which align to the ribosomal index. They are recorded but not used in later processing. `riboseq_data`/with-rrna/`sample-name`[.`note`].fastq.gz
    
* Reads not aligning to ribosomal sequences
    * **retained reads**. A fastq.gz file containing reads which do not align to the ribosomal index and are used in further processing. `riboseq_data`/without-rrna/`sample-name`[.`note`].fastq.gz
    
* Aligned reads
    * **sorted reads aligned to the genome**. A sorted bam file containing all alignments of reads to the genome. `riboseq_data`/without-rrna-mapping/`sample-name`[.`note`]Aligned.sortedByCoord.out.bam. Additionally, `riboseq_data`/without-rrna-mapping/`sample-name`[.`note`].bam, which is a symlink to the Aligned.sortedByCoord.out.bam file.
    
    * **aligned reads which map uniquely to the genome**. A sorted bam file containing all alignments of reads to the genome with multimapping reads filtered out. `riboseq_data`/without-rrna-mapping/`sample-name`[.`note`]-unique.bam
    
    * **unsorted reads aligned to the transcriptome**. An unsorted bam file containing all alignments of reads to the transcriptome. That is, the alignments are in "transcript space" coordinates. `riboseq_data`/without-rrna-mapping/`sample-name`[.`note`]Aligned.toTranscriptome.out.bam
    
    * **sorted reads aligned to the transcriptome**. A sorted bam file containing all alignments of reads to the transcriptome. That is, the alignments are in "transcript space" coordinates. `riboseq_data`/without-rrna-mapping/`sample-name`[.`note`].transcriptome.bam
    
    * **aligned reads which map uniquely to the transcriptome**. A sorted bam file containing all alignments of reads to the transcriptome with multimappers filtered out. That is, the alignments are in "transcript space" coordinates. This file is often not very useful since it filters reads which align to multiple transcripts. `riboseq_data`/without-rrna-mapping/`sample-name`[.`note`].transcriptome-unique.bam
    
* Metagene profiles
    * **metagene profiles**. A gzipped csv file containing the metagene profiles for all read lengths which occur in the uniquely-aligning reads. It includes the metagene profile around both the annotated translation initiation site and translation termination site. `riboseq_data`/metagene-profiles/`sample-name`[.`note`]-unique.metagene-profile.csv.gz
    
    * **periodicity estimations**. A gzipped csv file containing the Bayes factor estimates for all P-site offsets specified by the configuration, as well as information about the number of reads in the respective profile. `riboseq_data`/without-rrna-mapping/metagene-profiles/`sample-name`[.`note`]-unique.metagene-periodicity-bayes-factors.csv.gz
    
    * **estimated P-site offsets**. A gzipped csv file containing the selected P-site offset for each read length. All read lengths are included, even if the estimates do not meet the criteria specified in the configuration file. (The filtering occurs later.) `riboseq_data`/without-rrna-mapping/metagene-profiles/`sample-name`[.`note`]-unique.periodic-offsets.csv.gz

* ORF profiles
    * **unsmoothed ORF profiles**. A sparse [matrix market file](http://math.nist.gov/MatrixMarket/formats.html) containing the profiles for all ORFs. **N.B.** The matrix market format uses base-1 indices.  `riboseq_data`/orf-profiles/`sample-name`[.`note`]-unique.length-`lengths`.offset-`offsets`.profiles.mtx. 
    
    * **smoothed ORF profiles**. A sparse [matrix market file](http://math.nist.gov/MatrixMarket/formats.html) containing the smoothed profiles for all ORFs. **N.B.** The matrix market format uses base-1 indices.  `riboseq_data`/orf-profiles/`sample-name`[.`note`]-unique.length-`lengths`.offset-`offsets`.smooth.frac-`fraction`.rw-`reweighting-iterations`.profiles.mtx. 

Indices are also created for the bam files.

### Difference from paper

The fifth step of creating the base genome profile in the paper is "Everything except the 5' end of the remaining reads is removed." This profile is not explicitly constructed in the pipeline. The `sample-name`-unique.bam already contains the necessary information.



```python
create-orf-profiles test-chrI.fastq.gz c-elegans-test.yaml c-elegans-chrI --num-cpus 10
```

[Back to top](#toc)

<a id='predicting-translated-open-reading-frames'></a>

## Predicting translated open reading frames

The entire profile creation and translation prediction process can be run automatically using the `predict-translated-orfs` script. It reads most of the required paths from a [YAML](http://www.yaml.org/start.html) configuration file. Additionally, it automatically creates some of the output paths.

The script accepts a `--overwrite` flag. Unless this is given, then steps for which the output file already exists will be skipped.

It also accepts a `--do-not-call` flag. If this flag is given, then the commands below will be printed but not executed.

[Logging options](#logging-options) can also be given to this script.

[Parallel processing options](#parallel-processing-options) can be given to this script.

### Configuration file keys

The following keys are read from the configuration file. Keys with [`brackets`] are optional.

* `riboseq_data`. The base output location for all created files
* [`note`]. An optional string which will be added to all filenames. It should not contain spaces or any other special characters.
* `models_base`. The base path to the compiled models.

**N.B.** The models specified in the paper are included with the source distribution and compiled/pickled as part of the installation process. They are in the folders `/path/to/rp-bp/models/translated` and `/path/to/rp-bp/models/untranslated`. For a standard installation, the `models_base` should be `/path/to/rp-bp/models`.

#### Reference genome options
These options should be exactly the same as those used in the configuration file used to create the reference indices.

* `fasta`. The path to the reference genome sequence
* `genome_base_path`. The path to the output directory for the transcript fasta and ORFs
* `genome_name`. A descriptive name to use for the created files

#### Bayes factor estimate options
* [`chi_square_only`]. If this flag is in the config file with any value, then only the Rp-chi pipeline will be performed; namely, the translation models will not be fit to the data, and the posterior distributions will not be estimated.

* [`translation_iterations`]. The number of iterations to use for each chain in the MCMC sampling. The first half of the iterations are discarded as burn-in samples. All of the remaining samples are used to estimate the posterior distributions. (That is, we do not use thinning.) default: 200

#### Selecting predicted ORFs options

* [`min_bf_mean`]. The minimum value for the estimated Bayes factor mean to "predict" that an ORF is translated. This value is used in conjunction with both min_bf_mean and min_bf_likelihood.

 If max_bf_var is a positive value, then this is taken as a hard threshold on the estimated Bayes factor mean. ORFs must meet both the min_bf_mean and max_bf_var filters to be selected as "translated."

 If min_bf_likelihood is given, then this is taken as the boundary value; that is, an ORF is selected as "translated" if:

       [P(bf > min_bf_mean)] > min_bf_likelihood

 If both max_bf_var and min_bf_likelihood are None (null in YAML), then this is taken as a hard threshold on the mean for selecting translated ORFs.

 If both max_bf_var and min_bf_likelihood are given, then both filters will be applied and the result will be the intersection.
 
 default: 5

* [`max_bf_var`]. The maximum value value for the estimated Bayes factor variance to "predict" that an ORF is translated. ORFs must meet both the min_bf_mean and max_bf_var filters to be predicted. See the description for min_bf_mean for more details. default: null (i.e., this filter is not used by default)

* [`min_bf_likelihood`]. The minimum probability of the BF exceeding min_bf_mean to select an ORF as translated. See the description for min_bf_mean for more details. default: 0.5

* [`chisq_significance_level`]. For the chi-square test, this value is first Bonferroni corrected based on the number of ORFs which pass the smoothing filters. It is then used as the significance threshold to select translated ORFs. default: 0.01

#### Shared MCMC options
These affect the MCMC both for estimating metagene profile periodicity and ORF translation Bayes factors.
* [`seed`]. The random seed for the MCMC sampling. default: 8675309
* [`chains`]. The number of chains to use in the MCMC sampling. default: 2

### Input files
This script requires several files created during the previous steps in the pipeline, as well as a few external files.

* External files
    * **genome fasta file**. The genome fasta file. This is the same file used for `prepare-genome`.
    * **orfs**. The ORFs (gzipped bed12+ file) created by `prepare-genome`. It must be located at `genome_base_path`/transcript-index/`genome_name`.genomic-orfs.`orf_note`.bed.gz.
    * **models of translation**. Some compiled, pickled Stan model files must be located in the `models_base`/translated folder.
    * **models of lack of translation**. Some compiled, pickled Stan model files must be located in the `models_base`/untranslated folder.


* Metagene profiles
    * **metagene profiles**. A gzipped csv file containing the metagene profiles for all read lengths which occur in the uniquely-aligning reads. It includes the metagene profile around both the annotated translation initiation site and translation termination site. `riboseq_data`/metagene-profiles/`sample-name`[.`note`]-unique.metagene-profile.csv.gz
    
    * **periodicity estimations**. A gzipped csv file containing the Bayes factor estimates for all P-site offsets specified by the configuration, as well as information about the number of reads in the respective profile. `riboseq_data`/without-rrna-mapping/metagene-profiles/`sample-name`[.`note`]-unique.metagene-periodicity-bayes-factors.csv.gz
    
    * **estimated P-site offsets**. A gzipped csv file containing the selected P-site offset for each read length. All read lengths are included, even if the estimates do not meet the criteria specified in the configuration file. (The filtering occurs later.) `riboseq_data`/without-rrna-mapping/metagene-profiles/`sample-name`[.`note`]-unique.periodic-offsets.csv.gz

* ORF profiles
    * **unsmoothed ORF profiles**. A sparse [matrix market file](http://math.nist.gov/MatrixMarket/formats.html) containing the profiles for all ORFs. **N.B.** The matrix market format uses base-1 indices.  `riboseq_data`/orf-profiles/`sample-name`[.`note`]-unique.length-`lengths`.offset-`offsets`.profiles.mtx. 
    
    * **smoothed ORF profiles**. A sparse [matrix market file](http://math.nist.gov/MatrixMarket/formats.html) containing the smoothed profiles for all ORFs. **N.B.** The matrix market format uses base-1 indices.  `riboseq_data`/orf-profiles/`sample-name`[.`note`]-unique.length-`lengths`.offset-`offsets`.smooth.frac-`fraction`.rw-`reweighting-iterations`.profiles.mtx. 


### Output files
    
* Estimates
    * **the Bayes factor estimates**. A BED12+ file which contains the estimated values for all ORFs (which pass the thresholds mentioned above). The first 12 columns are valid BED12 entries that are simply copied from the `orfs` BED file.  `riboseq_data`/orf-predictions/`sample-name`[.`note`]-unique.length-`lengths`.offset-`offsets`.smooth.frac-`fraction`.rw-`reweighting-iterations`.bayes-factors.bed.gz. 
    
* Predictions
    * Rp-Bp predictions
        * **the ORFs**. A BED12+ file containing the ORFs in the final prediction set (the longest ORF for each stop codon which meets the filtering criteria). `riboseq_data`/orf-predictions/`sample-name`[.`note`]-unique.length-`lengths`.offset-`offsets`.smooth.frac-`fraction`.rw-`reweighting-iterations`.predicted-orfs.bed.gz. 
        
        * **the DNA sequences**. A fasta file containing the DNA sequences of the predicted ORFs. The fasta header matches the 'id' column in the BED files. `riboseq_data`/orf-predictions/`sample-name`[.`note`]-unique.length-`lengths`.offset-`offsets`.smooth.frac-`fraction`.rw-`reweighting-iterations`.predicted-orfs.dna.gz. 
        
        * **the protein sequences**. A fasta file containing the protein sequences of the predicted ORFs. The fasta header matches the 'id' column in the BED files. `riboseq_data`/orf-predictions/`sample-name`[.`note`]-unique.length-`lengths`.offset-`offsets`.smooth.frac-`fraction`.rw-`reweighting-iterations`.predicted-orfs.protein.gz. 
        
    * Rp-chi predictions
    
    Please note that Rp-chi does not use smoothing; therefore, the filenames do not include the smoothing options.
    
        * **the ORFs**. A BED12+ file containing the ORFs in the final prediction set (the longest ORF for each stop codon which meets the filtering criteria). `riboseq_data`/orf-predictions/`sample-name`[.`note`]-unique.length-`lengths`.offset-`offsets`.chisq.predicted-orfs.bed.gz. 
        
        * **the DNA sequences**. A fasta file containing the DNA sequences of the predicted ORFs. The fasta header matches the 'id' column in the BED files. `riboseq_data`/orf-predictions/`sample-name`[.`note`]-unique.length-`lengths`.offset-`offsets`.chisq.predicted-orfs.dna.gz. 
        
        * **the protein sequences**. A fasta file containing the protein sequences of the predicted ORFs. The fasta header matches the 'id' column in the BED files. `riboseq_data`/orf-predictions/`sample-name`[.`note`]-unique.length-`lengths`.offset-`offsets`.chisq.predicted-orfs.protein.gz. 


```python
predict-translated-orfs c-elegans-test.yaml c-elegans-chrI --num-cpus 10 --tmp /home/bmalone/tmp/
```

[Back to top](#toc)

<a id='logging-options'></a>

## Logging options

All of the driver scripts mentioned above, and many of the internal scripts as well, allow detailed specification of logging options on the command line. Interally, logging is handled using the standard python logging system. The following options are allowed.

The allowed logging levels are: `NOTSET`, `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`. Most of the scripts output possibly-useful information at the `INFO` level; some scripts also output `DEBUG` messages, but as the name indicates, they are typically only useful for tracking down problems which do not cause actual errors.

* `--log-file`. This option specifies a file to which logging statements will be written (in addition to stdout and  stderr, if specified). default: False

* `--log-stdout`. If this flag is present, then logging statements will be written to stdout (in addition to a file and stderr, if specified). default: False

* `--log-stderr`. If this flag is present, then logging statements will be written to stderr (in addition to a file and stdout, if specified). default: True

* `--logging-level`. If this value is specified, then it will be used for all logs. default: WARNING

* `--file-logging-level`. The logging level to be used for the log file, if specified. This option overrides `--logging-level`. default: NOTSET

* `--stdout-logging-level`. The logging level to be used for the stdout log, if specified. This option overrides `--logging-level`. default: NOTSET

* `--stderr-logging-level`. The logging level to be used for the stderr log, if specified. This option overrides `--logging-level`. default: NOTSET

[Back to top](#toc)

<a id='parallel-processing-options'></a>

## Parallel processing options

All of the scripts accept options for running code in parallel. Furthermore, the scripts are designed to seamlessly integrate with the [SLURM](http://slurm.schedmd.com/) scheduler. The following options are allowed. SLURM-specific options are specified by [`brackets`].

* `--num-cpus`. The number of CPUs to use. The definition of a "CPU" varies somewhat among the programs. For example, for STAR, these are actually threads. For many of the python scripts, this number is translated into the number of processes to spawn. None of the code parallelizes across machines, so the value should not be greater than the number of cores on the machine on which the programs are executed. When used with SLURM, this will be translated into an sbatch request like: "--ntasks 1 --cpus-per-task &lt;num-cpus&gt;". default: 1

* `--mem`. For STAR genome indexing, the amount of RAM to request. The rest of the programs do not use this value. When used with SLURM, this will be translated into an sbatch request like: "--mem=&lt;mem&gt;". default: 10G

* `--do-not-call`. If this flag is present, then the commands will not be executed (but will be printed). This can be useful to ensure paths in configuration files are correct.

* [`--use-slurm`]. If this flag is present, then the commands will be submitted to SLURM via sbatch. default: By default, each command is executed sequentially within the current terminal.

* [`--time`]. The amount of time to request. This will be translated into an sbatch request like: "--time &lt;time&gt;. default: 0-05:59

* [`--partitions`]. The partitions to request. This will be translated into an sbatch request like: "-p &lt;partitions&gt;". default: general (N.B. This value should be a comma-separated list with no spaces, for example: `--partitions general,long`)

* [`--no-output`]. If this flag is present, stdout will be redirected to /dev/null. This will be translated into an sbatch request like: "--output=/dev/null". default: If the flag is not present, then stdout will be directed to a log file with the job number. This corresponds to "--output=slurm-%J.out" in the sbatch call.

* [`--no-error`]. If this flag is present, stderr will be redirected to /dev/null. This will be translated into an sbatch request like: "--error=/dev/null". default: If the flag is not present, then stderr will be directed to a log file with the job number. This corresponds to "--error=slurm-%J.err" in the sbatch call.


```python
# This will submit the prepare-genome script to SLURM as a single job. That job
# will request 10 CPUs and 100G of RAM.

prepare-genome WBcel235.79.chrI.yaml --num-cpus 10 --mem 100G --use-slurm
```


```python
# This will submit each sample as a separate job to SLURM. Each submitted job will request 10 cpus
# and 100G of RAM. For example, if c-elegans-test.yaml specifies 5 samples in the riboseq_samples
# value, then 5 jobs will be submitted to SLURM, one for each sample.

process-all-samples c-elegans-test.yaml --tmp /home/bmalone/tmp/ --num-cpus 10 --mem 100G --use-slurm --logging-level INFO
```

[Back to top](#toc)