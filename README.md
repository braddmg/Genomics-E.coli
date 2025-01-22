# Pipeline for genomic analyses of Escherichia coli isolated from wastewaters
This pipeline analyzed the genomes of eight multidrug-resistant Escherichia coli strains isolated from wastewater treatment plants.

## Quality control with trimmomatic
Reads were filtered for quality control with [trimmomatic](https://github.com/timflutre/trimmomatic) 

```bash
for i in `ls -1 *_R1_001.fastq.gz | sed 's/_R1_001.fastq.gz//'`
do 
$i\_R1_001.fastq.gz $i\_R2_001.fastq.gz $i\_paired_R1.fastq.gz $i\_unpaired_R1.fastq.gz  $i\_paired_R2.fastq.gz $i\_unpaired_R2.fastq.gz ILLUMINACLIP:adapters-PE.fa:2:30:10:2:True LEADING:3 TRAILING:3 MINLEN:36
done
```
## Genome assembly 
Genome assembly was performed with SPAdes v3.14.1:
```bash
for i in `ls -1 *_R1.fastq.gz | sed 's/_R1.fastq.gz//'`
do 
spades.py -k 21,33,55,77,99 -t 12 --careful --pe1-1 $i_R1.fastq.gz --pe1-2  $i_ R2.fastq.gz -o ../../ASSEMBLY/
done
```
## Genome Quality Evaluation
Assembly metrics (e.g., N50, GC content) were calculated with QUAST, and genome completeness/contamination was assessed with CheckM2:. 

```bash
#Quast (contigs > 1000bp)
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
[GenFlow](https://github.com/braddmg/GenFlow) is a streamlined bioinformatic pipeline that combines several tools, including NCBI Datasets command-line, EDirect, Anvi’o, Prodigal, MAFFT, and FastTree, to provide users with an easy, one-command solution for performing phylogenomic and ANI analysis. We used used the tool to compare isolates alongside reference strains from phylogroups A, B1, B2, D1, E, and F1. Reference genome accession numbers are available in genomes.txt file in the repository.

```bash
#Genflow uses all fasta files in the directory 
GenFlow -g genomes.txt -G 0.8 -F 0.8 -t 32 -N
```
## MultiLocus sequence type 
MLST was performed with [mlst](https://github.com/tseemann/mlst) software.

```bash
mlst *.fasta
```

## Serotype and Antimicrobial resistance genes identification with ABRicate
We employed [ABRicate](https://github.com/tseemann/abricate) to predict the serotype of isolates by analyzing genes encoding the lipopolysaccharide (O) and flagellar (H) surface antigen genes. Additionally,  ABRicate was employed with the [AMRFinder](https://github.com/ncbi/amr) database (NCBI) to identify antibiotic resistance genes (ARGs).

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
Integron sequences were identified with [IntegronFinder](https://github.com/gem-pasteur/Integron_Finder). The regions spanning the identified integrons were extracted from the assembled genomes using [SAMtools ](https://www.htslib.org). 

```bash
#IntegronFinder
for i in `ls -1 *.fasta | sed 's/.fasta//'`
do
integron_finder --local-max --func-annot $i\.fasta
done

#Example to extract integron sequence in the A1539 genome
samtools faidx A1539.fa NODE_27_length_63054_cov_14.190061:21334-23845
```

The identified integron sequences were saved as FASTA files, annotated with [Anvi’o](https://anvio.org) using COG14 and Pfam databases, and subsequently analyzed with [MobMess](https://github.com/michaelkyu/MobMess) software. 

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
