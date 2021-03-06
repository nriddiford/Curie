# Tutorial to detect structual variants with SVDetect

# Table of Contents
* [About the tool](#about-the-tool)
  * [Linking](#linking-and-filtering)
  * [Filtering](#filtering)	
* [Input](#input)
  * [Preprocess bam files](#preprocess-bam-files)
  * [Genome file](#genome-file)
  * [Config files](#config-files)
* [SVDetect commands](#svdetect-commands)
  * [Linking and filtering](#linking-and-filtering)
    *[Bed file](#bed-file)
    *[SV file](#sv-file)
* [Output](#output)

# About the tool
[SVDetect](http://bioinformatics.oxfordjournals.org/content/26/15/1895.full.pdf) is a tool to identify genomic structural variations from paired-end and mate-pair sequencing data.
Applying both sliding-window and clustering strategies, it uses anomalously mapped read pairs to localise genomic rearrangements and classify them according to their type e.g.:
* Large insertions and deletions
* Inversions
* Duplications
* Balanced and unbalanced inter-chromosomal translocations

This is achieved through the following steps:

## Linking
Starting from a list of anomalously mapped paired-end reads, SVDetect uses a sliding-window strategy to identify all groups of pairs sharing a similar genomic location. The reference genome is divided into overlapping windows of fixed size, and each pair of windows can possibly form a link if at least one pair anchors them by its ends.

To each link connecting two genomic fragments, a certain number of features such as chromosomal location, number of pairs, orientation and order of the involved paired-ends are assigned.


### Filtering
The filtering procedure of SVDetect takes as input all links previously identified and uses user-defined filtering parameter values to call PEM clusters.


# Input

SVDetect talkes input in a number of forms, this protocol uses bam files generated by `bwa` and sorted and indexed using samtools. 

The first step in SVDetect is to regroup all pairs that are suspected to originate from the same SV.

## Preprocess bam files
**Important**: for clustering analysis, the mate_file must contain anomalously mapped paired-ends only. 
Correct mapped pairs (same chromosome, correct strand orientation and good range of insert size, etc.) have to be filtered out with a preprocessing step. 
For this we use a perl script for preprocessing pairs provided in the SVDetect package "BAM_preprocessingPairs.pl" (SAM/BAM file format only).  

The input consists of paired-ends mapped to the reference genome, and the output will contain pairs where either the orientation of pairs is incorrect and/or the distance between them is out of the typical range.

```perl /bioinfo/guests/nriddifo/bin/BAM_preprocessingPairs.pl <sample.sorted.bam>```

and for the reference: 

```perl /bioinfo/guests/nriddifo/bin/BAM_preprocessingPairs.pl <reference.sorted.bam>```

This will create a file containing the aberrant reads for each bam file: `sample.sorted.ab.bam` and `reference.sorted.bam`, which we then point to in config files.

**Important**: Mu and sigma levels used in the config file are generated by running `BAM_preprocessingPairs.pl`


## Genome file

We also need to create a file called `genome.len` with the number, name and length of each chromosome. This information can be found in the first two columns of the genome.fa.fai file (created by `samtools faidx genome.fa`). The columns are: chromosome ID (sequential integer starting from 1), chromosome name (the same as indicated in the mate_file), and the chromosome length in bases.

For Dmel_6.12:

```
[Chr ID]\t[Chr name]\t[Length]
1       2L      23513712
2       2R      25286936
3       3L      28110227
4       3R      32079331
5       4       1348131
6       X       23542271
7       Y       3667352
```

## Config files

We need to make a config file for both the sample (tumour) and reference samples that will be used for each step in the analysis. The [SVDetect manual](http://svdetect.sourceforge.net/Site/Manual.html) contains a thorough description of the options for each block. 

Call these `sample.sv.conf` and `reference.sv.conf`. 

`sample.sv.conf` example:
 

```{html}
<general>
input_format=bam 
sv_type=all
mates_orientation=RF
# Change the next 5 lines accordingly
read1_length=125
read2_length=125
mates_file=/path/to/sample.sorted.ab.bam										
cmap_file=/path/to/genome.len													
output_dir=/path/to/results														
tmp_dir=tmp
num_threads=2
</general>

<detection>
split_mate_file=1
window_size=6541
step_length=1635
</detection>

<filtering>
split_link_file=0
strand_filtering=1
order_filtering=1
insert_size_filtering=1
nb_pairs_threshold=5
nb_pairs_order_threshold=2
indel_sigma_threshold=3
dup_sigma_threshold=2
singleton_sigma_threshold=4
final_score_threshold=0.8
# Change the mu and sigma to values produced by BAM_preprocessingPairs.pl
mu_length=183
sigma_length=832
</filtering>

<bed>
<colorcode>
190,190,190 = 1,2 # grey
0,0,0       = 3,3 # black
0,0,255     = 4,4 # blue
0,255,0     = 5,5 # green
153,50,205  = 6,7 # purple
255,140,0  = 8,10 # orange
255,0,0     = 11,10000 # red
</colorcode>
</bed>

<compare>
# Change the next 3 lines accordingly
list_samples=sample.sorted,reference.sorted
list_read_lengths=125-125,125-125
file_suffix=.ab.bam.all.links.filtered
min_overlap=0.05
same_sv_type=1
# skip for now
circos_output=0
bed_output=1
sv_output=1
</compare>
```

The reference config file `reference.sv.conf` will look similar, but doesn't need the `<compare>` block. 

# SVDetect commands

Each different command uses the information we've entered in our `sample.sv.conf` and `reference.sv.conf` files.  
Below, I will discuss each of these in turn and provide quoted details from the [SVDetect sourceforge manual](http://svdetect.sourceforge.net/Site/Manual.html)  

## Linking and filtering

Main program for mapping all anomalous mapped paired-end reads onto a fragmented reference genome. First, the reference genome is partitioned into small genomic overlapped regions (typically twice the maximum distance insert size between ends). Each pair is then assigned to at least one possible pair of two chromosomal regions. Two genomic regions connected by the mapped ends of pairs is considered as a link. 
Redundant links are filtered out and the precise coordinates of the remaining unique links are determined. After a appropriate sorting procedure, the program proceeds to the union of close overlapped links.


Generation and filtering of links from the sample data:

```SVDetect linking filtering -conf sample.sv.conf```
```SVDetect linking filtering -conf reference.sv.conf```

The output file is tabulated output listing all of the chromosomal links (i.e. both inter- and intrachromosomal)

One file gives all the links `.all.links` and another gives the links after fiterering using the defined parameters `all.links.filtered`

Each column contains the follwoing information: 

1. Chromosome name of the first group of paired-end reads (=chromosome1)
2. Chromosome1 start coordinate of the link 
3. Chromosome1 end coordinate of the link
4. Chromosome name of the second group of paired-end reads (=chromosome2)
5. Chromosome2 start coordinate of the link
6. Chromosome2 end coordinate of the link
7. Number of mate-pairs in the chromosome link
8. Name list of the mate-pairs
9. Strand orientation list of the first group of paired-end reads
10. Strand orientation list of the second group of paired-end reads
11. Rank position of the read compared to its mate in the first group of paired-ends
(1 or 2, correspond to F3 or R3 for SOLiD data)
12. Rank position of the read compared to its mate in the second group of paired-ends
13. Read order of the first group of paired-ends
14. Read order of the second group of paired-ends according to the first group order
15. Sequencing start coordinates of the first group of paired-end reads
16. Sequencing start coordinates of the second group of paired-end reads

**Note** All positions are 1-based coordinates

The first few lines of the `.all.links` file I get for my `sample` are as follows: 

| 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 2L | 74 | 198 | 2L | 11968895 | 11969019 | 1 | HWI-D00405:129:C6KNAANXX:4:1215:15111:95983 | F | F | 2 | 1 | 1 | 1 | 74 | 11968895 |
| 2L | 112 | 236 | 2L | 2934154 | 2934278 | 2 | HWI-D00405:129:C6KNAANXX:4:1205:3258:73512,HWI-D00405:129:C6KNAANXX:4:1211:1294:50458 | F,F | F,F | 1,1 | 2,2 | 1,2 | 1,2 | 112,112 | 2934154,2934154 |

These describe two interchromosomal translocations on chromosome 2L. The first is supported by one read pair, and the second by two.  

## Filtering

The filtering procedure of SVDetect takes as input all identified links and uses user-defined filtering parameters to call PEM clusters. The **minimum number of paired-ends** is one of the most important filtering parameters to call a cluster. Setting a higher threshold here improves confidence in the detection of SVs.

Let's have a look at the filtering section of out `sample.sv.conf` file: 

```sed -n '/<filtering>/,/<\/filtering>/p' sample.sv.conf```

```{html}
<filtering>
split_link_file=0
strand_filtering=1
order_filtering=1
insert_size_filtering=1
nb_pairs_threshold=5
nb_pairs_order_threshold=2
indel_sigma_threshold=3
dup_sigma_threshold=2
singleton_sigma_threshold=4
final_score_threshold=0.8
mu_length=183
sigma_length=832
</filtering>
```

The only change I've made to the default setting (other than mu and sigma) really is the `-- nb_pairs_threshold` option. Here, I specify that there must a minimum of 5 pairs in a cluster (more conservative). Therefore, the two SVs identified in the `sample.all.links` file are not present in the `sample.all.links.filtered` file.

Filtering output contains the following fields: 

* 1-16. Fields with the same format of the linking procedure output. 
If order_filtering=1, the fields 8-16 are surrounded by "()" to highlight potential subgroups of reads. Pairs with features followed by :
  * "$" indicates pairs in a read (sub)group involved in an inconsistent orientation 
  * "*" indicates pairs in a read (sub)group with an inconsistent order 
  * "@" indicates pairs in a read (sub)group with an inconsistent insert size
* 17. Strand orientation feature of pairs after strand filtering running. If order_size_filtering=1 and/or insert_size_filtering=1,this field is replaced by the predicted type of intra-/intra-chromosomal SV
* 18. Mean separation distance if insert_size_filtering=1. 
* 19. Number of pairs after strand filtering / previous number of pairs
* 20. Balanced or unbalanced feature for order filtering
* 21. Number of pairs after order filtering / previous number of pairs
* 22. Number of pairs after insert size filtering / previous number of pairs
* 23. Subgroup coordinates of chromosome 1 if order_filtering=1
  * if UNBAL: predicted start and end coordinates of the concerned region for chromosome1
  * if BAL: start and end breakpoint coordinates for chromosome1
  The field is surrounded by "()" to highlight potential subgroups of reads
* 24. As 23. but for chromosome2
* 25. Score based on ratios from 18., 21. and 22. (best score=1)
* 26. Initial number of pairs (before applying filters)

The first 2 lines of `sample.all.links.filtered` contain: 

| 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 | 24 | 25 | 26 |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 2L | 112 | 236 | 2L | 2934154 | 2934278 | 2 | (HWI-D00405:129:C6KNAANXX:4:1205:3258:73512,HWI-D00405:129:C6KNAANXX:4:1211:1294:50458) | (F,F) | (F,F) | (1,1) | (2,2) | (1,2) | (1,2) | (112,112) | (2934154,2934154) | INVERSION | 2934042 | 2/2 | UNBAL | 2/2 | - | (1,112) | (2930617,2934154) | 1 | 2 |
| 2L | 304 | 428 | 2L | 11925560 | 11925684 | 3 | (HWI-D00405:129:C6KNAANXX:4:1116:19258:32222,HWI-D00405:129:C6KNAANXX:4:2212:15361:3516,HWI-D00405:129:C6KNAANXX:4:2304:9570:39544) | (F,F,F) | (F,F,F) | (2,2,2) | (1,1,1) | (1,2,3) | (1,2,3) | (304,304,304) | (11925560,11925560,11925560) | INVERSION | 11925256 | 3/3 | UNBAL | 3/3 | - | (1,304) | (11922023,11925560) | 1 | 3 |


In my file field 22 appears to be missing for some reason. I've added a `-` to reflect that here.


## Link comparison between datasets

Here we can use the `linkstocomapre` program to compare filtered links between two or more anomalously mapped mate-pair datasets and to identify common and sample-specific chromosomal links (like the usual sample/reference design).
Overlaps between link coordinates are used to filter the set of potential SVs.

`SVDetect links2compare -conf sample.sv.conf`

If comparing two samples like we are here this will generate the following files for each output type you specifiy in the `<compare>` block (prefix_suffix only): 
  * sample.reference_compared 
  * sample_compared
  * reference_compared 

We only asked for bed and sv output, so this produces 9 files in total.

### Bed file

Each line in the bed file corresponds to one mate-pair for intra-chromosomal links or one paire-end read for inter-chromosomal links:

1. Chromosome name pre-fixed with "chr"
2. Chromosome start coordinate (0-based)
3. Chromosome end coordinate
4. pair/read name (for reads /1 or /2 is added to distinguish the two ends)
5. score field set to 0
6. strand orientation (+ or -)
7. as 2.
8. as 3.
9. color associated to the number of pairs involved in the link. If the pair/read is involved in more than one link,
the associated color is set to the link having the highest number of pairs
10. =1 for pairs, = 2 for reads
11. read length(s)
12. relative start coordinates of reads (=0 for pairs)

First, sort the bedfile: 

`sort -k1,1V -k2,2n sample.bed > sample_sort.bed`

The first few lines of my sorted bed file look like this: 

| 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| chr2L | 111 | 2934278 | HWI-D00405:129:C6KNAANXX:4:1205:3258:73512 | 0 | + | 111 | 2934278 | 190,190,190 | 2 | 125,125 | 0,2934042 |
| chr2L | 111 | 2934278 | HWI-D00405:129:C6KNAANXX:4:1211:1294:50458 | 0 | + | 111 | 2934278 | 190,190,190 | 2 | 125,125 | 0,2934042 |
| chr2L | 226 | 351 | HWI-D00405:129:C6KNAANXX:4:1306:2535:23276/1 | 0 | + | 226 | 351 | 190,190,190 | 1 | 125 | 0 |
| chr2L | 226 | 351 | HWI-D00405:129:C6KNAANXX:4:1306:9513:42157/1 | 0 | + | 226 | 351 | 190,190,190 | 1 | 125 | 0 |
| chr2L | 303 | 11925684 | HWI-D00405:129:C6KNAANXX:4:1116:19258:32222 | 0 | + | 303 | 11925684 | 0,0,0 | 2 | 125,125 | 0,11925256 |

Another look at out `<bed><coverage>` block will give us info on the colours (comments added): 
	
```{html}
<bed>
<colorcode>
190,190,190 = 1,2 # grey
0,0,0       = 3,3 # black
0,0,255     = 4,4 # blue
0,255,0     = 5,5 # green
153,50,205  = 6,7 # purple
255,140,0  = 8,10 # orange
255,0,0     = 11,10000 # red
</colorcode>
</bed>	
```


Some of these can be viewed as tracks in IGV: 


`bedtools sort -sizeA -i sample_compared.bed > sample_compared_sorted.bed`
`bgzip sample_compared_sorted.bed`

### SV file 




## Depth-of-coverage analysis

Create a new config file for the sample only, called `sample.cnv.conf`: 

```{html}
<general>
input_format=bam
sv_type=all
mates_orientation=RF
read1_length=125
read2_length=125
mates_file=/path/to/sample.sorted.ab.bam										
cmap_file=/path/to/genome.len													
output_dir=/path/to/results	
tmp_dir=tmp
num_threads=2
</general>

<detection>
split_mate_file=1
window_size=30000^M
step_length=10000
mates_file_ref=/path/to/reference/original.bam
</detection>
```

**Important** make sure that the `-- mates_file_ref` option in the `<coverage>` block links to the _original_ bam file of the _reference_ (before processing with `BAM_preprocessingPairs.pl`)

`SVDetect cnv links2bed -conf sample.cnv.conf`

This produces a tabulated-text file listing coverage data of regions sized by the <window_size> parameter with the following fields:

1. Chromosome name
2. Chromosome start coordinate
3. Chromosome end coordinate
4. Average depth-of-coverage from the sample mate-pair data
5. Average depth-of-coverage from the reference mate-pair data
6. Log-ratio of 4. and 5. values
