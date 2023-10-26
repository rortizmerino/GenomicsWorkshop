# L3. DATA WRANGLING 2: SCRIPTING AND AUTOMATION

# automating a Variant Calling Workflow

In this lession we will make automated workflows for our previous learned variant calling commands by using shell scripts.

1. fastqc bash script.
2. fastqc sbatch script.
3. Variant Calling sbatch script.

## login on DelftBlue supercomputer

```console
[user@localcomputer ~]$ ssh <REPLACE-WITH-YOUR-NETID>@login.delftblue.tudelft.nl
```

## Getting the data

On DelftBlue every user has a scratch folder.

```console
[netid@login ~]$ cd /scratch/$USER
```

Copy the folder `dc_workshop` data to your scratch folder.

```console
[netid@login ~]$ cp -r /tudelft.net/staff-umbrella/WorkshopsData/dc_workshop .
```

And go into the folder dc_workshop

```console
[netid@login ~]$ cd dc_workshop
```

## fastqc bash script

make the bash script for the QC workflow.

```console
[netid@login ~]$ touch scripts/read_qc.sh dc_workshop
```

edit the new file and copy-past the scripts content from:
https://mvdb01.github.io/wrangling-genomics/05-automation/index.html

```console
Output
set -e
cd ~/dc_workshop/data/untrimmed_fastq/

echo "Running FastQC ..."
fastqc *.fastq*

mkdir -p ~/dc_workshop/results/fastqc_untrimmed_reads

echo "Saving FastQC results..."
mv *.zip ~/dc_workshop/results/fastqc_untrimmed_reads/
mv *.html ~/dc_workshop/results/fastqc_untrimmed_reads/

cd ~/dc_workshop/results/fastqc_untrimmed_reads/

echo "Unzipping..."
for filename in *.zip
    do
    unzip $filename
    done

echo "Saving summary..."
cat */summary.txt > ~/dc_workshop/docs/fastqc_summaries.txt
```

Open the script with nano and find replace (Alt+R) `~/dc_workshop` with `$ROOT`.  
Add these lines:

```console
ROOT="/scratch/$USER/dc_workshop"

source $ROOT/pathsloader
```

Add lines just before line 'mkdir -p $ROOT/results/fastqc_untrimmed_reads':

```console
if [ -d "$ROOT/results/fastqc_untrimmed_reads" ]
then
    echo "Directory $ROOT/results/fastqc_untrimmed_reads exists."
    rm -r $ROOT/results/fastqc_untrimmed_reads
fi
```

Add line at the end of the script:

```console
echo "doei doei!"
```

Run the script:

```console
[netid@login ~]$ bash scripts/read_qc.sh
```

## fastqc sbatch script

Make a sbatch script. We will use the read_qc.sh script as starting point.

```console
[netid@login ~]$ cp scripts/read_qc.sh scripts/read_qc.sbatch
```

Open with nano the new file:

```console
[netid@login ~]$ nano scripts/read_qc.sbatch
```

Add the following lines to the top of the file:

```console
#!/bin/sh

#SBATCH --job-name=fastqc
#SBATCH --partition=compute
#SBATCH --account=Education-EEMCS-Courses-genomicsworkshop
#SBATCH --reservation=genomicsworkshop
#SBATCH --time=00:05:00
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=1
#SBATCH --mem-per-cpu=1GB
#SBATCH –-mail-type=ALL
```

submit the fastqqc sbatch script.  

```console
[netid@login ~]$ sbatch scripts/read_qc.sbatch
```

Check the status of the job in the queue.  

```console
[netid@login ~]$ squeue -u $USER
```

Check after completion the job stats:

```console
[netid@login ~]$ seff "jobid"
```

## Excersice

Change (lower) different parameters like “time” and “mem-per-cpu” (one at the time) and submit the job again. Check with seff.

```console
#SBATCH --job-name=fastqc
#SBATCH --partition=compute
#SBATCH --account=Education-EEMCS-Courses-genomicsworkshop
#SBATCH --reservation=genomicsworkshop
#SBATCH --time=00:05:00
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=1
#SBATCH --mem-per-cpu=1GB
```

## Variant Calling sbatch script

Make a new file for the Variant Calling Pipeline.

```console
[netid@login ~]$ touch scripts/run_variant_calling.sbatch
```

Edit the new file:

```console
[netid@login ~]$ nano scripts/run_variant_calling.sbatch
```

Copy-paste script content into nano.
https://mvdb01.github.io/wrangling-genomics/05-automation/index.html

```console
set -e
cd ~/dc_workshop/results

genome=~/dc_workshop/data/ref_genome/ecoli_rel606.fasta

bwa index $genome

mkdir -p sam bam bcf vcf

for fq1 in ~/dc_workshop/data/trimmed_fastq_small/*_1.trim.sub.fastq
    do
    echo "working with file $fq1"

    base=$(basename $fq1 _1.trim.sub.fastq)
    echo "base name is $base"

    fq1=~/dc_workshop/data/trimmed_fastq_small/${base}_1.trim.sub.fastq
    fq2=~/dc_workshop/data/trimmed_fastq_small/${base}_2.trim.sub.fastq
    sam=~/dc_workshop/results/sam/${base}.aligned.sam
    bam=~/dc_workshop/results/bam/${base}.aligned.bam
    sorted_bam=~/dc_workshop/results/bam/${base}.aligned.sorted.bam
    raw_bcf=~/dc_workshop/results/bcf/${base}_raw.bcf
    variants=~/dc_workshop/results/bcf/${base}_variants.vcf
    final_variants=~/dc_workshop/results/vcf/${base}_final_variants.vcf 

    bwa mem $genome $fq1 $fq2 > $sam
    samtools view -S -b $sam > $bam
    samtools sort -o $sorted_bam $bam
    samtools index $sorted_bam
    bcftools mpileup -O b -o $raw_bcf -f $genome $sorted_bam
    bcftools call --ploidy 1 -m -v -o $variants $raw_bcf 
    vcfutils.pl varFilter $variants > $final_variants
   
    done
```

Open the script with nano and find replace (Alt+R) `~/dc_workshop` with `$ROOT`.  
Add these lines:

```console
ROOT="/scratch/$USER/dc_workshop"

source $ROOT/pathsloader
```

Add line at the end of the script:

```console
echo "doei doei!"
```

Add lines: (top of the script):

```console
#!/bin/sh
#SBATCH --job-name=fastqc
#SBATCH --partition=compute
#SBATCH --account=Education-EEMCS-Courses-genomicsworkshop
#SBATCH --reservation=genomicsworkshop
#SBATCH --time=00:05:00
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=1
#SBATCH --mem-per-cpu=1GB
#SBATCH --mail-type=ALL


module load 2022r2
module load bzip2
```

submit the job:

```console
[netid@login ~]$ sbatch scripts/run_variant_calling.sbatch
```

