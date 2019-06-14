# shart

*H*igh-throughput *A*nalysis of *R*eplication *T*iming 

A dockerfile and scripts for executing a semi-automatated repli-seq analysis pipeline

## what

This repository contains a dockerfile and scripts in order to generate replication timing profiles from a set of raw reads from sequencing of either early- and late-replicating DNA, or from DNA extracted from cells sorted for S or G1 DNA content.

The scripts for executing the pipeline are under the `scripts` which are added from this repository to the docker image during build time.

A docker image for executing these scripts can be built yourself or pulled from docker hub (vera/docker-4dn-repliseq).

## how

### example usage
```bash
# execute a step on data in the current directory
docker run -u $UID -w $PWD -v $PWD:$PWD:rw vera/shart <name_of_script> <args> 
````

### step-by-step workflow

#### setup
```bash
# pull the pre-built image, create and enter a container inside the directory with your data
docker run --rm -it -h d4r -u $UID:$(id -g) -w $PWD -v $PWD:$PWD:rw vera/shart

# define number of CPU threads to use for the pipeline
export NUMTHREADS=8
```
#### define your input files

```bash
# download example data
wget -cbre robots=off -np -nH --cut-dirs=3 -A 'g*' http://www.bio.fsu.edu/~dvera/share/repliseq/

# define early and late fastq files, here using sample data
E=$(ls *early*.fq.gz)
L=$(ls *late*.fq.gz)

index=bwaIndex_hg38/genome

```

#### execute workflow step by step

```bash
# clip adapters from reads with cutadapt
cfq=$(clip $E $L)

# align reads to genome with bwa
bam=$(align -i $index $cfq)
bstat=$(samstats $bam)

# filter bams by alignment quality and sort by position
sbam=$(filtersort $bam)
fbstat=$(samstats $sbam)

# remove duplicate reads
rbam=$(dedup $sbam)

# calculate RPKM bedGraphs for each set of alignments
bg=$(count $rbam)

# filter windows with a low average RPKM among libraries
fbg=$(filter $bg)

# calculate log2 ratios between early and late
l2r=$(log2ratio $fbg)

# quantile-normalize RT profiles to the average distribution
l2rn=$(normalize $l2r)

# loess-smooth profiles using a 300kb span size
l2rs=$(smooth -l 300000 -t $NTHREADS $l2rn)

organize
multiqc -f .

```
#### or use pipes
```bash
clip $E $L | align -i $index | filtersort | dedup | count | filter | log2ratio | normalize | smooth
organize
multiqc -f .
```
