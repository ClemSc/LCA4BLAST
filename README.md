# BLAST_2_LCA

BLAST_2_LCA performs Lowest Common Ancestor (LCA) estimation from Blast search results on Genbank database.

## Installation
BLAST_2_LCA is a single file Python 3 module that requires the Pandas library. Pandas can be installed with pip:

```console
pip install pandas
```
or conda
```console
conda install pandas
```
Then simply download BLAST_2_LCA.py from this repository and place it wherever you like. Make it executable, or run it with Python.
## Use
### Prerequisite.
BLAST_2_LCA takes as inputs a local blast search result file and NCBI taxonomy information files: nodes.dmp and names.dmp which are packaged in taxdump:
https://ftp.ncbi.nlm.nih.gov/pub/taxonomy/taxdump.tar.gz

Mind that your blast database must be built with taxid references. Here are some guidelines to build a suitable Blast search database:

1- Retrieve the accession2taxid files: https://ftp.ncbi.nlm.nih.gov/pub/taxonomy/accession2taxid/
— e.g. nucl_gb.accession2taxid.gz  and nucl_wgs.accession2taxid.gz if you intend to build a comprehensive nucleotide search DB.

2- Concatenate those two files, drop field names and keep only the accession number and taxid fields: 
```console
zcat nucl_gb.accession2taxid.gz | tail -n +2 > nucl.accession2taxid
zcat nucl_wgs.accession2taxid.gz | tail -n +2 >> nucl.accession2taxid
awk 'NR>1 {print $1,$3}' nucl.accession2taxid > tmp_acc2taxid
mv tmp_acc2taxid nucl.accession2taxid
 ```
3- Make the Blast search DB (e.g. from full nt database: https://ftp.ncbi.nlm.nih.gov/blast/db/FASTA/nt.gz)

```console
makeblastdb -in nt -dbtype nucl -out nt -taxid_map nucl.accession2taxid -parse_seqids
```

Hint: if you target a specific taxonomic group, you could use taxonize_gb (https://github.com/msabrysarhan/taxonize_genbank) to make a taxon restricted nt (or nr) dataset. E.g. for Nematoda (taxid = 6231).
```console
taxonize_gb --db nt --db_path nt.gz --taxid 6231 --nucl_gb_acc2taxid nucl_gb.accession2taxid.gz  --nucl_wgs_acc2taxid nucl_wgs.accession2taxid.gz --out Nematoda_nt
```

### Example Blast search and LCA estimation:
A typical Blast search looks like this, reads.fasta contains the queried sequences.

```console
blastn -query reads.fasta -task megablast -db nt -out blast_results.tsv -outfmt "6 qseqid saccver pident qcovs length evalue bitscore staxid" -num_threads 8 -evalue 1e-05
```

Adapt parameters values to your likings, but outfmt needs to be '6' and the following fields are mandatory: "qseqid pident length evalue staxid". Any extra fields will be shown in the final result, with values corresponding to the best hit for each surviving queries.

then perform LCA on the results:
```console
BLAST_2_LCA.py -t nodes.dmp -n names.dmp -i blast_results.tsv -o RESULTS_LCA.tsv -L 80 -H 95 -p 90 -l 350  -f "qseqid saccver pident qcovs length evalue bitscore staxid"
```

### BLAST_2_LCA parameters
#### Mandatory parameters:
- Input and output filenames are mandatory and are set with `-t/--nodes`, `-n/--names`, `-i/--input`, `-o/--output` (check the example command). Note: the input files can be provided compressed (e.g. gzipped).
- `-f/--fields` = Blast output fields. Use the same values and same order as in your Blast search command `-outfmt` without the leading "6", e.g. "qseqid saccver pident qcovs length evalue bitscore staxid"

#### Optional parameters (a default value will be used when not specified):
- `-H/-high_pident` = high similary threshold (percentage), default is 95
- `-L/--low_pident` = low similary threshold (percentage), default is 80.
  
  BLAST_2_LCA uses a high similarity and a low similarity threshold to decide whether a species level identification is relevant or not. Only queries for which at least one hit was found with percentage identity (pident) > H will get a chance to be identified to species. Otherwise, genus will be used as the lowest rank possible. Hits with pident < L are not used. In general, H should be set where the user estimate the species gap to be.

- `-p/--p_hits` = percentage of hits thresholds, default is 90.
  
Genbank is not perfect, and contains misidentified sequences. This parameter specifies the majority threshold over which a taxa should be accepted. E.g. if p = 90; then if 95% of the hits are assigned to _Megalothorax minimus_ and 5% to _Megalothorax willemi_, the query will be assigned to _Megalothorax minimus_. Else if only 85% of the hits are assigned to _M. minimus_ and 15% to _M. willemi_, then the query will be assigned to "_Megalothorax_".

- `-l/--length` = minimum alignment length, default is 350.
  
The minimum alignment length query/subject to be accepted.




