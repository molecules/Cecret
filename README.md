# Cecret

Named after the beautiful [Cecret lake](https://en.wikipedia.org/wiki/Cecret_Lake) 

Location: 40.570°N 111.622°W , 9,875 feet (3,010 m) elevation

![alt text](https://upload.wikimedia.org/wikipedia/commons/thumb/c/cb/Cecret_Lake_Panorama_Albion_Basin_Alta_Utah_July_2009.jpg/2560px-Cecret_Lake_Panorama_Albion_Basin_Alta_Utah_July_2009.jpg)

Cecret is a workflow developed by @erinyoung at the [Utah Public Health Laborotory](https://uphl.utah.gov/) for SARS-COV-2 sequencing with the [artic](https://artic.network/ncov-2019/ncov2019-bioinformatics-sop.html)/Illumina hybrid library prep workflow for MiSeq data with protocols [here](https://www.protocols.io/view/sars-cov-2-sequencing-on-illumina-miseq-using-arti-bffyjjpw) and [here](https://www.protocols.io/view/sars-cov-2-sequencing-on-illumina-miseq-using-arti-bfefjjbn). Built to work on linux-based operating systems. Additional config files are needed for cloud batch usage.

# Getting started

```
git clone https://github.com/UPHL-BioNGS/Cecret.git
```

To make your life easier, follow with

```
cd Cecret
git init
```

so that you can use `git pull` can be used for updates.

## Prior to starting the workflow

### Install dependencies
- [Nextflow](https://www.nextflow.io/docs/latest/getstarted.html) 
- [Singularity](https://singularity.lbl.gov/install-linux)

### Set default variables

Pangolin and Kraken2 images are both very large, and may cause an error if downloading normally. If this occurs, set `SINGULARITY_TMPDIR` to a location with a large amount of space and export the variable into the environment.

```
export SINGULARITY_TMPDIR=/scratch
```

### Create covid_samples.txt for easy file submissions (Optional)

This pipeline is meant to enable quick and easy file renaming for sample submission to public repositories. Essentially, the script parses a file named `covid_samples.txt` and assigns values based on the columns, with the first two coluns being the id used in the fastq file names (generally an accession number), and the second column being the id desired for submission to repositories, such as GenBank. Required values are the lab accession, submission id, and collection date.

Example covid_samples.txt file contents:
```
Lab_Accession	Submission_ID	Collection_Date	SRR
12345	UT-UPHL-12345	2020-08-22	SRR1
67890	UT-UPHL-67890	2020-08-22	SRR2
23456	UT-UPHL-23456	2020-08-22	SRR3
78901	UT-UPHL-78901	2020-08-18	SRR4
```
Where the files named `12345-UT-M03999-200822_S9_L001_R1_001.fastq.gz`, `12345-UT-M03999-200822_S9_L001_R2_001.fastq.gz` will be renamed `UUT-UPHL-12345.R1.fastq.gz` and `UT-UPHL-12345.R2.fastq.gz`. A consensus file will be duplicated and named `UT-UPHL-12345.consensus.fa`. GISAID and GenBank friendly multifasta files ready for submission are also generated. The GenBank multifasta uses the input file to create fasta headers like
```
>12345 [Collection_Date=2020-08-22][organism=Severe acute respiratory syndrome coronavirus 2][host=human][country=USA][isolate=SARS-CoV-2/human/USA/12345/2020][SRR=SRR1]
NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN
```

### Arrange the fastq.gz reads or specify reads paramater
```
directory
|-covid_samples.txt
|-Sequencing_reads
   |-Raw
     |-*fastq.gz
```

# Usage
With basic usage:
```
nextflow run Cecret.nf -c config/cecret.singularity.nextflow.config
```
Resuming a run to create a phylogenetic tree by setting the relatedness paramter to `"true"`
```
nextflow run Cecret.nf -c config/cecret.singularity.nextflow.config -resume --relatedness "true"
```

The main components of Cecret are:

- [seqyclean](https://github.com/ibest/seqyclean) - for cleaning reads
- [bwa](http://bio-bwa.sourceforge.net/) - for aligning reads to the reference
- [ivar](https://andersen-lab.github.io/ivar/html/manualpage.html) - for trimming primers, calling variants, and creating a consensus fasta
- [samtools](http://www.htslib.org/) - for QC metrics and sorting
- [fastqc](https://github.com/s-andrews/FastQC) - for QC metrics
- [bedtools](https://bedtools.readthedocs.io/en/latest/) - for depth estimation over amplicons
- [kraken2](https://ccb.jhu.edu/software/kraken2/) - for read classification
- [pangolin](https://github.com/cov-lineages/pangolin) - for lineage classification
- [mafft](https://mafft.cbrc.jp/alignment/software/) - for multiple sequence alignment (optional, relatedness must be set to "true")
- [snp-dists](https://github.com/tseemann/snp-dists) - for relatedness determination (optional, relatedness must be set to "true")
- [iqtree](http://www.iqtree.org/) - for phylogenetic tree generation (optional, relatedness must be set to "true")

## Final file structure
```
directory
|-covid_samples.txt
|-run_results.txt                       # information about the sequencing run that's compatible with legacy workflows
|-fastqc
| |-sample.fastqc.html
| |-sample.fastqc.zip
|-logs
| |-process_logs                        # for troubleshooting puroses
|-Sequencing_reads
| |-Raw
| | |-sample_S1_L001_R1_001.fastq.gz    # initial file
| | |-sample_S1_L001_R2_001.fastq.gz    # inital file
| |-QCed
|   |-sample_clean_PE1.fastq            # clean file
|   |-sample_clean_PE2.fastq            # clean file
|-covid
  |-bedtools
  | |-multicov.txt                      # coverage information for each amplicon
  |-bwa
  | |-sample.sorted.bam                 # aligned reads
  | |-sample.sorted.bam.bai             # indexes
  |-consensus
  | |-sample.consensus.fa               # consensus fasta file generated by ivar
  |-iqtree                              # optional: relatedness parameter must be set to true
  | |-iqtree.treefile
  |-ivar_trim
  | |-sample.primertrim.bam             # aligned reads after primer trimming
  |-kraken2
  | |-sample_kraken2_report.txt         # kraken2 report of the percentage of reads matching virus and human sequences
  |-mafft                               # optional: relatedness parameter must be set to true
  | |-mafft_aligned.fasta               # multiple sequence alignement generated via mafft
  |-pangolin
  | |-sample
  |   |-lineage_report.csv              # identification of pangolin lineages
  |-samtools_coverage
  | |-bwa
  | | |-sample.cov.hist                 # histogram of coverage for aligned reads
  | | |-sample.cov.txt                  # tabular information of coverage for aligned reads
  | |-trimmed
  |   |-sample.cov.trim.hist            # histogram of coverage for aligned reads after primer trimming
  |   |-sample.cov.trim.txt             # tabular information of coverage for aligned reads after primer trimming
  |-samtools_flagstat
  | |-bwa
  | | |-sample.flagstat.txt             # samtools stats for aligned reads
  | |-trimmed
  |   |-sample.flagstat.trim.txt        # samtools stats for trimmed reads
  |-samtools_stats
  | |-bwa
  | | |-sample.stats.txt                # samtools stats for aligned reads
  | |-trimmed
  |   |-sample.stats.trim.txt           # samtools stats for trimmed reads
  |-snp-dists                           # optional: relatedness parameter must be set to true
  | |-snp-dists                         # file containing a table of the number of snps that differ between any two samples
  |-submission_files                    # optional: is only created if covid_samples.txt exists
  | |-run_id.genbank_submission.fasta   # multifasta for direct genbank
  | |-run_id.gisaid_submission.fasta    # multifasta file bulk upload to gisaid
  | |-sample.consensus.fa               # renamed consensus fasta file
  | |-sample.genbank.fa                 # fasta file with formatting and header including metadata for genbank
  | |-sample.gisaid.fa                  # fasta file with header for gisaid
  | |-sample.R1.fastq.gz                # renamed raw fastq.gz file
  | |-sample.R2.fastq.gz                # renamed raw fastq.gz file
  |-summary
  | |-sample.summary.txt
  |-trimmed
  | |-sample.primertrim.sorted.bam      # aligned reads after primer trimming and sorting
  |-variants
  | |-sample.variants.tsv               # list of variants identified via ivar and corresponding scores
  |-summary.txt                         # comma separated file with coverage, depth, number of failed amplicons, and number of "N" information
```

# Adjustable Paramters

All workflow parameters can be adjusted in a config file or on the command line as follows:

```
nextflow run Cecret.nf -c config/cecret.singularity.nextflow.config --outdir new_directory
```

Adjustable workflow paramaters with default values:
```
reads = current_directory + '/Sequencing_reads/Raw'
artic_version = 'V3'
year = '2020'
outdir = current_directory
sample_file = current_directory/covid_samples.txt
relatedness = false
```
For use with staphb/seqyclean:1.10.09 container run with singularity
```
seqyclean_contaminant_file="/Adapters_plus_PhiX_174.fasta"
```
For use with staphb/ivar:1.2.2_artic20200528 container run with singularity
```
primer_bed = "/artic-ncov2019/primer_schemes/nCoV-2019/${params.artic_version}/nCoV-2019.bed"
reference_genome = "/artic-ncov2019/primer_schemes/nCoV-2019/${params.artic_version}/nCoV-2019.reference.fasta"
gff_file = "/reference/GCF_009858895.2_ASM985889v3_genomic.gff"
amplicon_bed = "/artic-ncov2019/primer_schemes/nCoV-2019/${params.artic_version}/nCoV-2019_amplicon.bed"
```
For use with staphb/kraken2:2.0.8-beta_hv container run with singularity
```
kraken2_db="/kraken2-db"
```

### Other useful options
To create a report, use `-with-report` with your nextflow command.

# Directed Acyclic Diagrams (DAG)
### Full workflow
![alt text](images/Cecret_workflow.png)

![alt text](https://uphl.utah.gov/wp-content/uploads/New-UPHL-Logo.png)
