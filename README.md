# Pipeline for genomic analyses of Escherichia coli isolated from wastewaters


## MobMess networks
Using MobMess, plasmid networks are generated and classified into backbone, compound, fragment, and maximal clusters. For more information about MobMess see the [Paper](https://www.nature.com/articles/s41564-024-01610-3):

The following command processes the plasmid.fasta file containing all plasmid sequences obtained from [PlasMidScope](https://plasmid.deepomics.org/database/plasmid), filters contigs < 1000 bp, annotates them with Anvi’o (using COG14 and Pfam databases), and creates MobMess networks.
Files used in the analysis can be found in the repository
```bash
