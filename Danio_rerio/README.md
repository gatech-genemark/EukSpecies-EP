# Species: _Danio_rerio_
Alex Lomsadze, Tomas Bruna  
Georgia Institute of Technology  
2019  
## Project setup
Project is set in bash shell.  

Setup environment on GT cluster as:  
```
umask 002

base="/storage3/w/alexl/EukSpecies"
species="Danio_rerio"

export PATH="$base/bin:$PATH"
export base="$base/$species"
cd $base
if [ "$(pwd)" != "$base" ]; then echo "error, folder not found: $base"; fi
```
Create core folders  
```
cd $base
mkdir -p arx annot data mask
```
### Assembly
Download genomic sequence and reformat it:  
 * simplified FASTA defline with a first word in defline as a unique sequence ID
 * select only nuclear DNA (exclude organelles)
 * set sequence in all uppercase

When possible use genomic sequence from NCBI.  
Match sequence ID in FASTA file with sequence ID in annotation file.  
Use ID from annotation.  
Keep IDs in the file "list.tbl".  
First column in the table is sequence ID and second column is annotation ID.  

Description of assembly is at https://www.ncbi.nlm.nih.gov/assembly/GCF_000002035.6/  
```
cd $base/arx
mkdir ensembl refseq

cd $base/arx/refseq
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/002/035/GCF_000002035.6_GRCz11/GCF_000002035.6_GRCz11_genomic.fna.gz
gunzip GCF_000002035.6_GRCz11_genomic.fna.gz

grep '^>' GCF*.fna > deflines.refseq
grep '^>' GCA*.fna > deflines.genbank
cat deflines.refseq | grep -v "^>NW" | grep -v "^>NC_002333.2" | cut -b2- | awk '{print $1 "\t" $7}' | tr -d ',' > ../list_refseq.tbl
cat deflines.genbank | grep -v "^>KZ" | grep -v "^>KN" | cut -b2- | awk '{print $1 "\t" $5}' | tr -d ',' > ../list_genbank.tbl

gzip GCF_000002035.6_GRCz11_genomic.fna
```
### Annotation
```
cd $base/arx/ensembl
wget ftp://ftp.ensembl.org/pub/release-97/gff3/danio_rerio/Danio_rerio.GRCz11.97.gff3.gz
gunzip Danio_rerio.GRCz11.97.gff3.gz

gff_to_gff_subset.pl  --in Danio_rerio.GRCz11.97.gff3  --out tmp_annot.gff3  --list ../list_genbank.tbl  --col 2  --v
echo "##gff-version 3" > annot.gff3
probuild --stat_fasta --seq ../../data/genome.fasta | cut -f1,2 | tr -d '>' | | grep -v '^$' | awk '{print "##sequence-region  " $1 "  1 " $2}' >> annot.gff3
cat tmp_annot.gff3 | grep -v gff-version  >> annot.gff3
rm  tmp_annot.gff3

# check
/home/tool/gt/bin/gt  gff3validator annot.gff3
# reformat
/home/tool/gt/bin/gt  gff3  -force  -tidy  -sort  -retainids  -checkids  -o tmp_annot.gff3  annot.gff3
mv tmp_annot.gff3  annot.gff3
enrich_gff.pl --in annot.gff3 --out ensembl.gff3 --cds
gff3_to_gtf.pl ensembl.gff3  ensembl.gtf
# check
compare_intervals_exact.pl --f1 ensembl.gff3  --f2 ensembl.gtf
/home/braker/src/eval-2.2.8/validate_gtf.pl -c annot.gtf

mv ensembl.gff3   ../../annot/
mv ensembl.gtf    ../../annot/

# separate pseudo
cd $base/arx/ensembl
select_pseudo_from_nice_gff3.pl annot.gff3 pseudo.gff3
mv pseudo.gff3 ../../annot/

# masking coordinates
cd $base/annot/
soft_fasta_to_3 < ../data/genome.fasta.masked | awk '{print $1 "\tsoft_masking\trepeat\t" $2+1 "\t" $3+1 "\t.\t.\t.\t." }' > mask.gff
```
###  APPRIS
Data from http://appris.bioinfo.cnio.es
```
# download
wget http://apprisws.bioinfo.cnio.es/pub/releases/2019_07.v29/datafiles/danio_rerio/GRCz10/appris_data.appris.txt

# get PRINCIPAL transcript ID's
cat ../annot/annot.gtf | grep -E -o 'gene:\w+' | sort | uniq  | sed s'/^gene://' > tmp_gene_names
fgrep -f tmp_gene_names  appris_data.appris.txt | grep PRINCIPAL | cut -f3 | sed 's/^/transcript:/' > appris.tbl
rm tmp_gene_names
../../bin/select_by_trascript_id_from_gtf.pl  appris.tbl  ../annot/annot.gtf  appris.gtf
rm appris.tbl

# If multiple PRINCICAL transcripts are annotated per gene, then select the longest 
# In case of equal length, select the first one
../../bin/get_longest_cds_gene_set.pl --in appris.gtf  --out appris.tbl -v
../../bin/select_by_trascript_id_from_gtf.pl  appris.tbl  ../annot/annot.gtf  appris.gtf
rm appris.tbl

mv appris.gtf ../annot/
```
### Dealing with incomplete CDS

* The following script:
    * Flags partial CDS
    * Removes extra start and stops in the enriched annotation
    * Splits the annotation into files with:
        * Complete/incomplete transcripts
        * Complete/incomplete genes. Gene is considered to be incomplete if at least one of its transcripts is incomplete.

Assumes that annot.gtf is the enriched version of annotation with **incorrect starts and stops being part of partial CDS segments**.

```bash
cd $base/annot
flagPartialCDS.py ../arx/ensembl/Danio_rerio.GRCz11.97.gtf annot.gtf --incompleteTranscriptsOutput incompleteTranscripts.gtf \
    --completeTranscriptsOutput completeTranscripts.gtf --fullOutput annot_fixed_partial.gtf --completeGenesOutput completeGenes.gtf \
    --incompleteGenesOutput incompleteGenes.gtf
```

Select complete genes in APPRIS.

```bash
flagPartialCDS.py ../arx/ensembl/Danio_rerio.GRCz11.97.gtf appris.gtf --fullOutput appris_fixed_partial.gtf --completeGenesOutput \
    appris_completeGenes.gtf --incompleteTranscriptsOutput /dev/null --completeTranscriptsOutput /dev/null --incompleteGenesOutput /dev/null
```
