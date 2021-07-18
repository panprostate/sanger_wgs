---
editor_options: 
  markdown: 
    wrap: 72
---

# Sanger Pipeline

<!-- badges: start -->

<!-- badges: end -->

Pipeline used by PPCG for Whole Genome Sequencing quality control,
allignment and variant calling.

# Requirements

## Libraries and Tools

-   `conda`: <https://docs.conda.io/en/latest/>
-   `samtools`: <http://www.htslib.org/>
-   `fastsplit`: <https://github.com/supernifty/fastqsplit>
-   `pigz`: <https://zlib.net/pigz/>
-   `biobambam2`: <https://github.com/gt1/biobambam2>
-   `gatk`: <https://gatk.broadinstitute.org/hc/en-us>
-   `cgpNgsQc`: <https://github.com/cancerit/cgpNgsQc>
-   `cgpmap-2.0.3`: <https://github.com/cancerit/dockstore-cgpmap>
-   `cgpwgs-1.1.2`: <https://github.com/cancerit/dockstore-cgpwgs>
-   `udocker` or `singularity`

## Reference Files

-   For `cgpmap`: <https://github.com/cancerit/dockstore-cgpmap/wiki>
-   For `cgpwgs`:
    <https://github.com/cancerit/dockstore-cgpwgs/wiki/Reference-archives>
-   In this page, download all reference files related to Human
    `GRCh37`, although `cgpwgs` pipeline would primarily need
    `core_ref_GRCh37d5.tar.gz`

# Pre-alignment

If starting from `BAM` files, use `samtools` to sort, split in read1 and
2 and transform to `FASTQ`

    samtools collate -u -O $FILE | samtools fastq -1 ${FILE%.bam}_r1.fastq -2 ${FILE%.bam}_r2.fastq -0 /dev/null -s /dev/null -n

## Split by lane

`FASTQ` files were splitted by lane using `fastqsplit`. Note
that`fastqsplit` requires gziped files (`pigz` offers parallel fast
zipping) and illumina `FASQ` version 1.8 or higher. For `FASTQ` file
version older than Casanova 1.8 a modified version of `fastqsplit` can
be used (<https://github.com/eddieimada/fastqsplit>).

The guideline for `FASTQ` file naming is
`{sample}_{flowcell}_{barcode}_L00{lane}_R1.fastq.gz`

## Convertion to BAM

In this step, files were converted to BAM with read group info. For each
of these per-lane FASTQ files, the `fastqtobam` in `biobambam2` package
was used to convert to unmapped BAM files.

-   Options for fastqtobam:
-   gz: set to 1 if input fastqs are gzipped, 0 otherwise.
-   namescheme: set to pairedfiles if fastqs are of paired-end
    sequencing
-   RGCN: Sequencing centre name, can set to your institution's name or
    abbreviation
-   RGID: Read group ID, set to an unique ID for this lane. Try to make
    sure it's unique across all lanes of all samples
-   RGSM: Sample ID, an unique ID across all of your samples
-   RGPL: Sequencing platform. We accept ILLUMINA only.
-   RGLB: Sequencing library ID. Set to a string of a library ID of your
    choice
-   I: Input fastq file name, specified it one more time for the second
    file of a pair.

example:

    /path/to/biobambam2/bin/fastqtobam gz=1 namescheme=pairedfiles \
    RGCN=AU \
    RGID=AU:ID_1 \
    RGSM=TestS1 \
    RGPL=ILLUMINA \
    RGLB=lib-Test \
    I=TestS1_L001_R1.fastq.gz \
    I=TestS1_L001_R2.fastq.gz \
    > TestS1_L001.bam

## Merging to single BAM

Step for merging BAM files into a single BAM file per sample If there
were multiple per-lane BAM files, they were merged to a single unmapped
per-sample BAM file using `bamcat` in `biobambam2` or any other tools,
such as Picard\`s MergeSamFiles
(<https://broadinstitute.github.io/picard/>).
```
    example:
    /path/to/biobambam2/bin/bamcat \
    md5=1 \
    md5filename=TestS1.merge.bam.md5 \
    I=TestS1_L001.bam \
    I=TestS1_L002.bam \
    > TestS1.bam
```
Or
```
    $ module load picard
    $ java -jar $EBROOTPICARD/picard.jar MergeSamFiles I=../fastqtobam/TESTS-H5F3GDSXX-ATTACTCG+TAATATTA_L001.bam I=../fastqtobam/TESTS-H5F3GDSXX-ATTACTCG+TATTCTTA_L001.bam I=../fastqtobam/TESTS-H5F3GDSXX-ATTACTCG+TAATCTTA_L001.bam O=TESTS.unmapped.bam
```

_*Alternative Tools for Merging*_

As 2021,`biobambam2` is outdated and unsupported. `biobambam2` depends
on the library `libmaus2` that is outdate and unsupported as well.
Alternativelly, `gatk FastqToSam` can be used to a obtain same results.

To merge two files
`PCAWG.faeb4dd6-68d9-4bed-82e8-d12adca6b28c_6_r1.fastq.gz`,
`PCAWG.faeb4dd6-68d9-4bed-82e8-d12adca6b28c_6_r2.fastq.gz`, here the
example code:
```
        gatk FastqToSam --SEQUENCING_CENTER  WCM
                        --READ_GROUP_NAME    faeb4dd6-68d9-4bed-82e8-d12adca6b28c_6
                        --SAMPLE_NAME        faeb4dd6-68d9-4bed-82e8-d12adca6b28c
                        --PLATFORM           ILLUMINA
                        --LIBRARY_NAME       sample_name_lib
                        -F1                  PCAWG.faeb4dd6-68d9-4bed-82e8-d12adca6b28c_6_r1.fastq.gz
                        -F2                  PCAWG.faeb4dd6-68d9-4bed-82e8-d12adca6b28c_6_r2.fastq.gz
                        -O
```
**Make sure to use `ILLUMINA` as platform argument, since that is required in later steps to succesfully validate samples.**

## Validation

In this stea `cgpNgsQc` inspects the metadata table and assign UUIDs for each donor and sample.

This step need a tab-delimited metdata file with the following header:
```
column 1: Donor_ID 
column 2: Tissue_ID 
column 3: is_normal 
column 4: is_normal_for_donor 
column 5: Sample_ID column 6: relative_file_path
```
Create a tab-separated file with a content like this example
(`test_input.tsv`)
```
        Donor_ID Tissue_ID is_normal is_normal_for_donor Sample_ID relative_file_path
        TestP0  WG  Y   Y   TestS0  ./TestS0.bam
        TestP0  Primary N       TestS1  ./TestS1.bam
```

Note that table needs to list for each donor a normal + a  tumour sample.


`cgpNgsQc` package comes in a docker container, but it can be executed with
`udocker` (<https://github.com/indigo-dc/udocker>), if `docker` is unavailable for your system.

Below an Example code for loading the docker image to `udocker`:
```
        udocker pull quay.io/wtsicgp/docker-cgp-ngs-qc
        udocker create quay.io/wtsicgp/docker-cgp-ngs-qc:latest
        udocker name <CONTAINER ID for quay.io/wtsicgp/docker-cgp-ngs-qc:latest> <name>
```

The CONTAINER ID can be found with `udocker ps`.

Below is an example code to run the validation step using `udocker`.

`` udocker run -v `pwd`:`pwd`-w`pwd`udocker_cgp-ngs-qc validate_sample_meta.pl -in test_input.tsv -out test_output.tsv -format tsv ``

Sucessfull exit will return a `test_output.tsv` file with UUIDs that can be used by `cgpmap` for alignment.

# Allignment

Example code is well documented in
`https://github.com/cancerit/dockstore-cgpmap#other-uses`. For
Australian data,  version 2.0.3 was used, and `ds-wrapper.pl` was run
instead of `ds-cgpmap.pl`.

Below is an example code for an Australian sample using `singularity`.

```
    singularity exec -i --bind
     {input_dir}:/mnt/in,{output_dir}:/mnt/out,{reference_dir}:/mnt/reference,{temporary_dir}:/mnt/tmp
    --workdir {working_dir} --home {`home` under \`working_dir}:/home
    [path/to/dockstore-cgpmap-2.0.3.simg] ds-wrapper.pl -reference
    /mnt/reference/core_ref_GRCh37d5.tar.gz -bwa_idx
    /mnt/reference/bwa_idx_GRCh37d5.tar.gz -sample {sample UUID} TestS1.bam
```
-   --workdir: workspace directory
-   --home: a directory where all result files are stored

Once alignment is done, it's recommended to change the resulting BAM
file to a more suitable name, such as `TestS1.mapped.bam\`.

# Variant calling

Running `cgpwgs`: 

Example code Australian samples using Singularity

    singularity exec -i --bind
     {input_dir}:/mnt/in/,{output_dir}:/mnt/out,{reference_dir}:/mnt/reference
    --workdir {working_dir} --home {working_dir}/home:home
    [path/to/dockstore-cgpwgs-1.1.2.simg] ds-wrapper.pl
    -reference="/mnt/reference/core_ref_GRCh37d5.tar.gz"
    -annot="/mnt/reference/VAGrENT_ref_GRCh37d5_ensembl_75.tar.gz"
    -snv_indel="/mnt/reference/SNV_INDEL_ref_GRCh37d5.tar.gz"
    -cnv_sv="/mnt/reference/CNV_SV_ref_GRCh37d5_brass6+.tar.gz"
    -subcl="/mnt/reference/SUBCL_ref_GRCh37d5.tar.gz"
    -tumour="/mnt/in/{sample_mapped_bam}"
    -normal="/mnt/in/{matched_normal_mapped_bam}"
    -exclude="NC_007605,hs37d5,GL%" -species="human" -assembly="GRCh37d5"
    -cavereads="350000"

-   --workdir: workspace directory
-   --home: a directory where all result files are stored
