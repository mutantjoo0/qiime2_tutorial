# QIIME2 tutorial
By:
[Dr Umer Zeeshan Ijaz][http://userweb.eng.gla.ac.uk/umer.ijaz] Last Updated: 27/08/2019

Assuming you have organized the samples in separate folder with the folder name specifying the sample names, and within each folder, there is a "Raw" folder containing the paired-end forward and reverse files. 


**Step 1**: All you have to do is to run the following one liners to create the fictious barcodes and save them as `sample_metadata.tsv` file. Please make sure you 


```bash
d="/PATH_TO_FOLDER/"; 
t=$(ls $d | wc -l);
paste <(ls $d) <(perl -le 'sub p{my $l=pop @_;unless(@_){return map [$_],@$l;}return map { my $ll=$_; map [@$ll,$_],@$l} p(@_);} @a=[A,C,G,T]; print join("", @$_) for p(@a,@a,@a,@a,@a,@a,@a,@a);' | awk -v k=$t 'NR<=k{print}') | awk 'BEGIN{print "sample-id\tbarcode-sequence\n#q2:types\tcategorical"}1' > sample_metadata.tsv
```

```diff
+ In the above, the perl one liner is used to generate a unique 8bp barcode for every sample folder. Below, we are generating a 3bp as an example.
perl -le 'sub p{my $l=pop @_;unless(@_){return map [$_],@$l;}return map { my $ll=$_; map [@$ll,$_],@$l} p(@_);} @a=[A,C,G,T]; print join("", @$_) for p(@a,@a,@a)'
AAA
AAC
AAG
AAT
ACA
ACC
ACG
ACT
AGA
AGC
AGG
AGT
ATA
ATC
ATG
ATT
CAA
CAC
CAG
CAT
CCA
CCC
CCG
CCT
CGA
CGC
CGG
CGT
CTA
CTC
CTG
CTT
GAA
GAC
GAG
GAT
GCA
GCC
GCG
GCT
GGA
GGC
GGG
GGT
GTA
GTC
GTG
GTT
TAA
TAC
TAG
TAT
TCA
TCC
TCG
TCT
TGA
TGC
TGG
TGT
TTA
TTC
TTG
TTT
```




**Step 2**: Generate barcodes
```bash
(for i in $(ls $d); do bc=$(awk -v k=$i '$1==k{print $2}' sample_metadata.tsv); bioawk -cfastx -v k=$bc '{print "@"$1" "$4"\n"k"\n+";for(i=0;i< length(k);i++){printf "#"};printf "\n"}' $d/$i/Raw/*_R1.fastq ; done) > barcodes.fastq
```


**Step 3**: Assemble forward files
```bash
(for i in $(ls $d); do bioawk -cfastx '{print "@"$1" "$4"\n"$seq"\n+\n"$qual}' $d/$i/Raw/*_R1.fastq ; done) > forward.fastq
```

**Step 4**: Aseemble reverse files
```bash
(for i in $(ls $d); do bioawk -cfastx '{print "@"$1" "$4"\n"$seq"\n+\n"$qual}' $d/$i/Raw/*_R2.fastq ; done) > reverse.fastq
```

**Step 5**: Zip all the files and move them to
```bash
gzip *.fastq
mkdir emp-paired-end-sequences; mv *.gz emp-paired-end-sequences/.
```

**Step 6**: Enable Qiime2 on Orion cluster
```bash
export PATH=/home/opt/miniconda2/bin:$PATH
source activate qiime2-2019.7
```

See if qiime is working properly:
```bash
qiime --help


Usage: qiime [OPTIONS] COMMAND [ARGS]...

  QIIME 2 command-line interface (q2cli)
  --------------------------------------

  To get help with QIIME 2, visit https://qiime2.org.

  To enable tab completion in Bash, run the following command or add it to
  your .bashrc/.bash_profile:

      source tab-qiime

  To enable tab completion in ZSH, run the following commands or add them to
  your .zshrc:

      autoload bashcompinit && bashcompinit && source tab-qiime

Options:
  --version   Show the version and exit.
  --help      Show this message and exit.

Commands:
  info                Display information about current deployment.
  tools               Tools for working with QIIME 2 files.
  dev                 Utilities for developers and advanced users.
  alignment           Plugin for generating and manipulating alignments.
  composition         Plugin for compositional data analysis.
  cutadapt            Plugin for removing adapter sequences, primers, and
                      other unwanted sequence from sequence data.
  dada2               Plugin for sequence quality control with DADA2.
  deblur              Plugin for sequence quality control with Deblur.
  demux               Plugin for demultiplexing & viewing sequence quality.
  diversity           Plugin for exploring community diversity.
  emperor             Plugin for ordination plotting with Emperor.
  feature-classifier  Plugin for taxonomic classification.
  feature-table       Plugin for working with sample by feature tables.
  fragment-insertion  Plugin for extending phylogenies.
  gneiss              Plugin for building compositional models.
  longitudinal        Plugin for paired sample and time series analyses.
  metadata            Plugin for working with Metadata.
  phylogeny           Plugin for generating and manipulating phylogenies.
  quality-control     Plugin for quality control of feature and sequence data.
  quality-filter      Plugin for PHRED-based filtering and trimming.
  sample-classifier   Plugin for machine learning prediction of sample
                      metadata.
  taxa                Plugin for working with feature taxonomy annotations.
  vsearch             Plugin for clustering and dereplicating with vsearch.
```

**Step 7**: Import the data according to the recommendations given at https://docs.qiime2.org/2019.7/tutorials/importing/#importing-seqs

```bash
qiime tools import \
  --type EMPPairedEndSequences \
  --input-path emp-paired-end-sequences \
  --output-path emp-paired-end-sequences.qza
```

**Step 8**: Demultiplex and visualise 

```bash
qiime demux emp-paired --p-no-golay-error-correction --i-seqs emp-paired-end-sequences.qza --m-barcodes-file sample_metadata.tsv --m-barcodes-column barcode-sequence --o-per-sample-sequences demux.qza --o-error-correction-details demux-details.qza
```

```bash
qiime demux summarize --i-data ./demux.qza  --o-visualization ./demux.qzv
qiime tools export --input-path demux.qzv --output-path output
```

**Step 9**: Now try Dada2

```bash
qiime dada2 denoise-paired --i-demultiplexed-seqs demux.qza --p-trim-left-f 0 --p-trim-left-r 0 --p-trunc-len-f 240 --p-trunc-len-r 200 --p-n-threads 0 --o-table table.qza --o-representative-sequences rep-seqs.qza --o-denoising-stats denoising-stats.qza --verbose
```

**Step 10**: Create phylogenetic tree
```bash
qiime phylogeny align-to-tree-mafft-fasttree --i-sequences rep-seqs.qza --o-alignment aligned-rep-seqs.qza --o-masked-alignment masked-aligned-rep-seqs.qza --p-n-threads 0 --o-tree unrooted-tree.qza --o-rooted-tree rooted-tree.qza
```
[http://userweb.eng.gla.ac.uk/umer.ijaz]: http://userweb.eng.gla.ac.uk/umer.ijaz "Dr Umer Zeeshan Ijaz"
