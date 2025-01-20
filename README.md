# Pipeline for genomic analyses of Escherichia coli isolated from wastewaters
This pipeline was employed to analyze the genome of eight multidrug-resistant Eschericha coli isolated from wastewater treatment plants. 

## Quality control with trimmomatic
Raw data may be filtered to keep only high quility reads. For this we employed [trimmomatic](https://github.com/timflutre/trimmomatic) 

```bash
for i in `ls -1 *_R1_001.fastq.gz | sed 's/_R1_001.fastq.gz//'`
do 
$i\_R1_001.fastq.gz $i\_R2_001.fastq.gz $i\_paired_R1.fastq.gz $i\_unpaired_R1.fastq.gz  $i\_paired_R2.fastq.gz $i\_unpaired_R2.fastq.gz ILLUMINACLIP:adapters-PE.fa:2:30:10:2:True LEADING:3 TRAILING:3 MINLEN:36
done
```
## Genome assembly 
For genome assembly we employed SPAdes v3.14.1 with the next command:
```bash
for i in `ls -1 *_R1.fastq.gz | sed 's/_R1.fastq.gz//'`
do 
spades.py -k 21,33,55,77,99 -t 12 --careful --pe1-1 $i_R1.fastq.gz --pe1-2  $i_ R2.fastq.gz -o ../../ASSEMBLY/
done
```
## Evaluation of genome quality
To generate assembly metrics such as N50, L50, size, GC content and other, we employed QUAST. Additionally we used checkm2 to evaluate the completeness and contamination of assembled genomes. 

```bash
#Quast only evaluating contigs longer than 1000bp
for i in `ls -1 *.fasta | sed 's/.fasta//'`
do 
/opt/bioinf/quast-4.6.0/quast-4.6.0/quast.py -L -s $i\.fasta -o QUAST2/$i\ --min-contig 1000

#Checkm2
for i in `ls -1 *.fasta | sed 's/.fasta//'`
do
checkm2 predict --threads 32 --input $i\.fasta --output-directory checkm2/$i
done
```
## Phylogenomic analysis with GenFlow 
[GenFlow](https://github.com/braddmg/GenFlow) is a streamlined bioinformatic pipeline that combines several tools, including NCBI Datasets command-line, EDirect, Anvi’o, Prodigal, MAFFT, and FastTree, to provide users with an easy, one-command solution for performing phylogenomic and ANI analysis. In our case we employed the tool to compare our isolates with other reference strains belonging to phylogroups A, B1, B2, D1, E and F1. The genomes.txt file with the reference accession numbers is available in the repository. 
Additionally, GenFlow uses anvio to remove contigs shorter than 1000bp and produce a new fasta file.

```bash
#Genflow uses all fasta files in the directory 
GenFlow -g genomes.txt -G 0.8 -F 0.8 -t 32 -N
```
## MultiLocus sequence type 
MLST was assesed with [mlst](https://github.com/tseemann/mlst) software.

```bash
mlst *.fasta
```

## Serotype and Antimicrobial resistance genes identification with ABRicate
We employed [ABRicate](https://github.com/tseemann/abricate) to identify the potential serotype of isolates based on lipopolysaccharide (O) and flagellar (H) surface antigen genes and to identify antibiotic resistance genes (ARGs) with the AMRFinder database (NCBI).

```bash
for i in `ls -1 *.fasta | sed 's/.fasta//'`
do 
abricate --db ecoh  $i\.fasta > $i\.ecoh
abricate --db ncbi  $i\.fasta > $i\.ncbi
done
```

## Plasmid identification
We reconstructed and classified plasmid sequences with [MOB-suite](https://github.com/phac-nml/mob-suite).

```bash
for i in `ls -1 *.fasta | sed 's/.fasta//'`
do
mob_recon --infile $i\.fa --outdir $i\_mob/
done
```

## Comparative Analysis of Integron Sequences 
In this step, integron sequences were identified with [IntegronFinder](https://github.com/gem-pasteur/Integron_Finder). The regions spanning the identified integrons were extracted from the assembled genomes using [SAMtools ](https://www.htslib.org). 

```bash
#IntegronFinder
for i in `ls -1 *.fasta | sed 's/.fasta//'`
do
integron_finder --local-max --func-annot $i\.fasta
done

#Example to extract integron sequence in the A1539 genome
samtools faidx A1539.fa NODE_27_length_63054_cov_14.190061:21334-23845
```

Those integron sequences were saved as fasta files and then annnotated with [Anvi’o](https://anvio.org) using COG14 and Pfam and subsequently analyzed with [MobMess](https://github.com/michaelkyu/MobMess) software. 

```bash
anvi-script-reformat-fasta integrons.fasta \
                           -o integrons.fa \
                           --seq-type NT --simplify-names --prefix integrons
anvi-gen-contigs-database -f integrons.fa -o integrons.db
anvi-run-hmms -c integrons.db
anvi-export-gene-calls --gene-caller prodigal -c integrons.db -o integrons-gene-calls.txt
done

anvi-run-ncbi-cogs -T 32 --cog-version COG14 --cog-data-dir /work/databases/anvio/COG_2014 -c integrons.db
anvi-run-pfams -T 32 --pfam-data-dir /work/bmendoza/Tesis/Data/plasmids/anvio/Pfam_v32 -c integrons.db
anvi-export-functions --annotation-sources COG14_FUNCTION,Pfam -c integrons.db -o integrons-cogs-and-pfams.txt
done

#Visualization 
contigs=integrons_000000000001,integrons_000000000002,integrons_000000000003,integrons_000000000004,integrons_000000000005,integrons_000000000006
mobmess visualize -s integrons.fa -a integrons-cogs-and-pfams.txt -g integrons-gene-calls.txt -o figura/ -T 32 --contigs $contigs --align-blocks-height 1
```
