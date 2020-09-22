![.](./ONT_logo.png "Oxford Nanopore Technologies")

**************************

# 1. Introduction

### Overview:

This pipeline manages a pore-c workflow starting from raw fastq files and converting
them to standard file formats for use by downstream tools. The steps involved are:

* Pre-processing a reference genome or draft assembly to generate auxiliary files used in downstream analyses
* Creating virtual digests of the genome
* Filtering the raw reads to remove any that might break downstream tools
* Align against a reference genome
* Processing results to filter spurious alignments, detect ligation junctions and assign fragments. The results are stored in a [parquet](http://parquet.apache.org/) table for downstream processing.
* Converting the results to the following formats:
  - [pairs format](https://github.com/4dn-dcic/pairix/blob/master/pairs_format_specification.md)
  - [cooler format](https://mirnylab.github.io/cooler/)
  - [hic medium format](https://github.com/aidenlab/juicer/wiki/Pre#medium-format-most-common)
  - [salsa2 bed format](https://github.com/marbl/SALSA)


# 2. Getting started

In most cases, it is best to pre-install conda before starting. All other dependencies will be installed automatically when running the pipeline for the first time.

### Requirements:

This pipline requires a computer running Linux (Ubuntu 16). >64Gb of memory would be recommended. The pipeline has been tested on minimal server installs of these operating systems.

Most software dependencies are managed using *conda*. To install conda, please install [miniconda3](https://conda.io/miniconda.html) and refer to installation [instructions](https://conda.io/projects/conda/en/latest/user-guide/install/index.html).
You will need to accept the license agreement during installation and we recommend that you allow the Conda installer to prepend its path to your .bashrc file when asked.

```
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
```

Check if the conda has successfully installed

```
conda -h
```

If conda has installed correctly, you should see the follow output.
If you do not see the below output, you may need to close and reopen your terminal.

```
$ conda
usage: conda [-h] [-V] command ...

conda is a tool for managing and deploying applications, environments and packages.

Options:

positional arguments:
  command
    clean        Remove unused packages and caches.
    config       Modify configuration values in .condarc. This is modeled
                 after the git config command. Writes to the user .condarc
                 file ($HOME/.condarc) by default.
    create       Create a new conda environment from a list of specified
                 packages.
..............
```

---

#### Installation:

Clone this git repository to the location where you want to run your analysis and create the conda environment that will be used to run the pipeline

```
git clone https://github.com/nanoporetech/Pore-C-Snakemake.git
cd pore-c-snakemake
## Creates environment and the dependencies will install automatically
conda env create
conda activate pore_c_snakemake
```
**Note** before you run any of the snakemake commands below  you need to make sure that you've run `conda activate pore_c_snakemake`.

---

# 3. Usage

#### Testing:

Test data is included in the `.test` subfolder (`git-lfs` is required to download them). To run the tests use

    snakemake --use-conda  test -j 4 --config=output_dir=results.test

The results of the test run will appear in the `results.test` directory.

#### Configure workflow:

The pipeline configuration is split across several files:


    *  `config/config.yaml` - A yaml file containing settings for the pipeline. Input data is specified in the following tab-delimited files.
    *  `config/basecall.tsv` - Metadata and locations of the pore-c sequencing run fastqs.
    *  `config/references.tsv` - Locations of the draft/scaffold/reference assemblies that the pore-c reads will be mapped to.
    *  `config/phased_vcfs.tsv` - [Optional] The location of phased vcf files that can be used to haplotag poreC reads.


#### Execute workflow:

Test your configuration by performing a dry-run via

    snakemake --use-conda -n

Execute the workflow locally via

    snakemake --use-conda --cores $N

using `$N` cores or run it in a cluster environment via

    snakemake --use-conda --cluster qsub --jobs 100

or

    snakemake --use-conda --drmaa --jobs 100

in combination with any of the modes above.  See the [Snakemake documentation](https://snakemake.readthedocs.io/en/stable/executable.html) for further details.



#### Workflow targets

The pipeline defines several targets that can be speficied on the command line:

*  **all:**  The default target which builds the `pore_c` contact and concatemer parquet files under the `merged_contacts` directory.
*  **cooler:**  Builds a multi-resolution `.mcool` file.
*  **pairs:**  Builds a `pairix`-indexed pairs file.
*  **juicer:**  Builds a `.hic` file compatible with the `juicebox` suite of tools.
*  **salsa:**  Builds a `.bed` file for use with the `salsa2` scaffolding tool.
*  **mnd:**  Builds a `.mnd.txt` file compatible with the `3d-dna` scaffolding tool [experimental].


To build the files for a particular target:

    snakemake --use-conda -j 8 <target>



# 4. Output files [TODO: needs to be updated]
Once the pipeline has run successfully you should expect the following files in the output directory:

*  **`align_table/`:**
    *  `*.at.catalog.yaml` - an [intake](https://intake.readthedocs.io/en/latest/) catalog containing metadata about the alignment table.
    *  `*.at.alignment.parquet` - records containing all alignment information in parquet format.
    *  `*.at.alignment_summary.csv` - alignment summary table.
    *  `*.at.overlap.parquet` - table of the overlaps between alignments and fragments.
    *  `*.at.read.parquet` - per-read alignment statistics (contacts per-read etc).
    *  `*.at.read_summary.csv` - summary stats of input reads, such as read N50.
*   **`basecall/`:**
    *   `*.rd.catalog.yaml` - an [intake](https://intake.readthedocs.io/en/latest/) catalog containing read metadata.
    *   `*.rd.pass.fq.gz` - basecalls that have passed filtering.
    *   `*.rd.read_metadata.parquet` - a table of per-read statistics.
    *   `*.rd.summary.csv` - a table of aggregate statistics for the reads.
*   **`mapping/`:**
    *   `*.read_sort.bam` - bam alignment file sorted by read name.
*   **`matrix/`**
    *   `*.matrix.catalog.yaml` - an [intake](https://intake.readthedocs.io/en/latest/) catalog containing metadata about the aggregate matrix.
    *   `*.matrix.coo.csv.gz` - aggregate read counts in the format 'bin1_id,bin2_id,count' - suitable for use with `cooler load` the bin width for this set by the `*base*` matrix resolution in the config file.
    *   `*.matrix.cool` - the aggregate contact counts in [cooler](https://mirnylab.github.io/cooler/) format
    *   `*.matrix.counts.mcool` - a multi-resolution cool file.
*   **`pairs/`:**
    *   `*.pairs.pairs.gz` - contains fragment position and fragment pairs in [pairs format](https://github.com/4dn-dcic/pairix/blob/master/pairs_format_specification.md).
*   **`refgenome/`:**
    *   `*.rg.metadata.csv` - chromosome metadata in csv format.
    *   `*.rg.chromsizes`- reference genome chromosome lengths
    *   `*.rg.fa.gz` - reference genome compressed with bgzip
    *   `*.rg.fa.gz.fai` - samtools indexed reference genome
    *   `*..rg.fa.gz.bwt` - bwa index reference genome
*   **`virtual_digest/`:**
    *   `*.vd.fragments.parquet` - A table containing the intervals generated by the virtual digest.
    *   `*.vd.digest_stats.csv` - virtual digest aggregate statistics
*   **`assembly/`:**
    *   `*.salsa2.bed` - *optional* a bed file compatible with the [salsa2](https://github.com/marbl/SALSA) scaffolding tool.
*   **`juicebox/`:**
    *   `*.hicRef` - *optional* a [restriction site format](https://github.com/aidenlab/juicer/wiki/Pre#restriction-site-file-format) file.
    *   `*.hic.txt` - *optional* a [hic medium format](https://github.com/aidenlab/juicer/wiki/Pre#medium-format-most-common) file of pairwise contacts.


#### License and Copyright:
© 2019 Oxford Nanopore Technologies Ltd.

Bioinformatics-Tutorials is distributed by Oxford Nanopore Technologies under the terms of the MPL-2.0 license.
