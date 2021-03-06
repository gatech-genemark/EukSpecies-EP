# EukSpecies  
Genome and annotation preparation protocol for eukaryotic projects.  
  
Only shared components are described in this document.  
Species specific commands are in species README files.  

### Installation  
```
git clone https://github.gatech.edu/GeneMark/EukSpecies-EP.git

cd EukSpecies
cd bin

# follow installation instructions in "bin" README file
```
### Setup  
Project is set in bash shell.  

### Genome sequence  
Download genomic sequence and reformat it:  
 * Input sequence should be in FASTA format.
 * Unique sequence ID should have no white space symbols in it.
 * Simplify FASTA definition line (defline). First word in defline should be a unique sequence identifier.
 * Select only nuclear DNA sequences from genome (exclude organelles).
 * Set all sequence letters into uppercase.
  
Use genomic sequence from GenBank when possible. GenBank website and sequence accession IDs are usually more stable than genome project websites. On opposite, annotation is more frequently up-to-date at genomic project locations. Download most reliable annotation.  
* Match sequence ID in FASTA file with sequence ID in annotation file.  
* Use ID from annotation.  
* Keep information about genome sequence ID and annotation sequence ID in the file "list.tbl".  
* First column in the "list.tbl" table is sequence ID and second column is annotation ID.  
