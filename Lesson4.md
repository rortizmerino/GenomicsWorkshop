# L4. GENOME ASSEMBLY AND EVALUATION

Genome assembly is a method for re-constructing a genome from a large number of (short- or long-) DNA fragments (reads) when no reference genome is available.

During this lesson you will use sequencing reads from different technologies:

1) short reads from Illumina
2) and long read from Oxford Nanopore

## login on DelftBlue supercomputer

```console
[user@localcomputer ~]$ ssh <REPLACE-WITH-YOUR-NETID>@login.delftblue.tudelft.nl
```

## Getting the data

On DelftBlue every user has a scratch folder.

```console
[netid@login ~]$ cd /scratch/$USER
```

Copy the folder `asm_workshop` data to your scratch folder.

```console
[netid@login ~]$ cp -r /tudelft.net/staff-umbrella/WorkshopsData/asm_workshop/ .
```

And go into the folder asm_workshop

```console
[netid@login ~]$ cd asm_workshop
```

## Trimming and Quality Control

During this session we will use Illumina Short reads and Oxford Nanopore long reads. We will do QC on both data sets and compare them in one script.

```console
[netid@login ~]$ sbatch scripts/1.runTrimmomatic_Fastqc.sbatch
```

Download both (illumina and ont) fastqc results to your local computer and compare the results.

```console
[user@localcomputer ~]$ mkdir ~/Desktop/asm_workshop/fastqc_html

[user@localcomputer ~]$ scp YOUR-NETID@login.delftblue.tudelft.nl:/scratch/$USER/asm_workshop/results/fastqc_trimmed_reads/*html ~/Desktop/asm_workshop/fastqc_html/

[user@localcomputer ~]$ scp YOUR-NETID@login.delftblue.tudelft.nl:/scratch/$USER/asm_workshop/results/fastqc_ont_reads/*html ~/Desktop/asm_workshop/fastqc_html/
```

## *De Novo* assembly

*De Novo* assembly is the process of merging short sequencing reads into contiguous sequences (contigs).

Now that we checked and trimmed the Illumina Paired End library we are ready to assemble it.

### SPAdes Genome Assembler

We will use the ‘SPAdes Genome Assembler’. SPAdes takes as input paired-end reads, mate-pairs and single (unpaired) reads in FASTA and FASTQ format. In a first step SPAdes will do a read error corrction and use these in the iterative short-read genome assembly.

### Illumina Paired End

First inspect the script we are going to use.

```console
[netid@login ~]$ less scripts/2.runSpades_PE.sbatch
```

And now submit the job:

```console
[netid@login ~]$ sbatch scripts/2.runSpades_PE.sbatch
```

SPAdes created folder spades_pe in folder results. Inspect the content of the folder.

```console
[netid@login ~]$ ls -al results/spades_pe

[netid@login ~]$ less results/spades_pe/contigs.fasta

[netid@login ~]$ grep "NODE" results/spades_pe/contigs.fasta | less

[netid@login ~]$ grep "NODE" results/spades_pe/contigs.fasta | wc -l
```

### Illumina Paired End + Mate-pair

In most assembly projects multiple libraries with different insert sizes are used.

Here we will add an 2.5 kb Mate pair library.

Due to the library preparation the read orientation of these libraries are different: PE: → ← , MP: ← → OR → ←

First inspect the script we are going to use.

```console
[netid@login ~]$ less scripts/3.runSpades_PE_MP.sbatch
```

And now submit the job:

```console
[netid@login ~]$ sbatch scripts/3.runSpades_PE_MP.sbatch
```

SPAdes created folder spades_pe_mp in folder results. Inspect the content of the folder.

```console
[netid@login ~]$ ls -al results/spades_pe_mp

[netid@login ~]$ less results/spades_pe_mp/scaffolds.fasta

[netid@login ~]$ grep "NODE" results/spades_pe_mp/scaffolds.fasta | less

[netid@login ~]$ grep "NODE" results/spades_pe_mp/scaffolds.fasta | wc -l
```

### Oxford Nanopore

Recent new single molecule technologies like PacBio and Oxford Nanopore are promising. Here we will assemble data from NCBI Bioproject PRJDB9013, which was used for Benchmarks of de novo assemblers for bacterial genomes. The data we will use is E. coli str. K12 substr. MG1655 with identifier DRR198814 which was sequenced on a MinION from Oxford Nanopore.

We will use Flye assembler for long reads.
https://github.com/fenderglass/Flye

First inspect the script we are going to use.

```console
[netid@login ~]$ less scripts/4.runFlye.sbatch
```

And now submit the job:

```console
[netid@login ~]$ sbatch scripts/4.runFlye.sbatch
```

Flye created folder flye_ont in folder results. Inspect the content of the folder.

```console
[netid@login ~]$ ls -al results/flye_ont

[netid@login ~]$ less results/flye_ont/assembly_info.txt

[netid@login ~]$ less results/flye_ont/assembly.fasta
```

### Compare the three assemblies with Quast

To inspect the results we can use QUAST http://quast.sourceforge.net/quast to evaluate the assemblies.

First inspect the script we are going to use.

```console
[netid@login ~]$ less scripts/5.runQuast.sbatch
```

And now submit the job:

```console
[netid@login ~]$ sbatch scripts/5.runQuast.sbatch
```

Quast created a folder quast in results. Download this folder to your local computer to inspect the results.

```console
[user@localcomputer ~]$ cd ~/Desktop/asm_workshop

[user@localcomputer ~]$ scp -r YOUR-NETID@login.delftblue.tudelft.nl:/scratch/$USER/asm_workshop/results/quast .
```

### Polishing with illumina short reads

flye does an error correction step by mapping back the long reads to the assembly and create a consensus sequence from this mapping. We can further improve the assembly by doing the same but now with Illumina reads and Pilon.
https://github.com/broadinstitute/pilon/wiki

First inspect the script we are going to use.

```console
[netid@login ~]$ less scripts/6.runPolish.sbatch
```

And now submit the job:

```console
[netid@login ~]$ sbatch scripts/6.runPolish.sbatch
```

```console
[netid@login ~]$ ls -al results/flye_polished
```

flye_ont_polished.fasta is the Illumina polished assembly. And flye_ont_polished.changes contains all corrections made by Pilon.

```console
[netid@login ~]$ less results/flye_polished/flye_ont_polished.changes
```

### Assembly Error rate

Since we are using noisy reads to do a De Novo assembly we would like to know how this will workout for the assembly.

We could align the assembly to the reference sequence we have to inspect the sequence similarity.

For this we will use the tool dnadiff from the MUMmer package. https://github.com/garviz/MUMmer

First inspect the script we are going to use.

```console
[netid@login ~]$ less scripts/7.runDnaDiff.sh
```

And now run the job on the login node:

```console
[netid@login ~]$ bash scripts/7.runDnaDiff.sh
```

dnadiff creates for every alignment a report. We are interested in the AvgIdentity in the report. With 'cat' and 'grep' we can extract this information from the three reports.

```console
[netid@login ~]$ cat ../results/dnadiff/*.report | grep -A4 Alignments
```

The first output is from the Flye assembly, the second from the polished Flye assembly and the last one from the Spades assembly.

### Visualise the assembly graphs with Bandage

Download the assembly graphs from the spades assembly (results/spades_pe/assembly_graph.fastg) and the Flye assembly (results/flye_ont/assembly_graph.gfa) to your local computer and op them with Bandage.

### Genome annotation

Now that we have assembled our genome we would like to do an annotation. For prokaryotic genomes Prokka is a very good and fast annoation pipeline.
https://github.com/tseemann/prokka

We will run Prokka on the spades_pe and the polished Flye assemblies.

First inspect the script we are going to use.

```console
[netid@login ~]$ less scripts/8.runProkka_spades.sbatch

[netid@login ~]$ less scripts/9.runProkka_flye.sbatch
```

And now run both jobs on the login node:

```console
[netid@login ~]$ bash scripts/8.runProkka_spades.sbatch

[netid@login ~]$ bash scripts/9.runProkka_flye.sbatch
```


