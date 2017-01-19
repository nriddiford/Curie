# Tutorial to detect copy number variation using CNV-Seq

# Table of Contents
* [About the tool](#about-the-tool)
* [Input](#input)
  * [Hits files](#hits-file)
* [Run CNV-Seq](#cnv-seq)
* [Output](#output)
  * [Selecting for high/low coverage regions](#selecting-for-high-and-low-coverage-regions)
  * [Plotting](#plotting)
* [To do](#to-do)

# About the tool

CNV-Seq allows the detection of copy number variation using NGS data.

# Input

In order to reduce the noise around repeat regions in the genome, first filter reads with a mapping quality > 4: 

`samtools view -b -q 4 sample.bam > sample.qfilt.bam`

## Hits file
We only need to provide two "hits" files for CNV-Seq to work, one for the sample, and one for the reference. 

These can be extracted as follows: 

`samtools view -F 4 sample.qfilt.bam | perl -lane 'print "$F[2]\t$F[3]"' > sample.qfilt.hits` 

This creates a two column, tab delimited file, with the second column giving the corresponding 1-based leftmost mapping position of a read.

```
2L	1
2L	1
2L	4
2L	4
2L	4
2L	4
2L	4
2L	4
2L	4 
```

Next, to select only fully assembled chromosomes run the following script. This will output filtered files eg `sample.qfilt.hits.filt`:

```{perl}
#/usr/bin/perl
use strict;
use warnings;
use feature qw /say/;

my @files = @ARGV;

my @keys = qw / 2L 2R 3L 3R 4 X Y /;
my %filter;
$filter{$_} = 1 for (@keys);

for my $f (@files){
open my $in, '<', $f or die $!;
open my $out, '>', "$f\.filt" or die $!;

    while(<$in>){
        chomp;
        my @split = split;
        print $out "$_\n" if $filter{$split[0]};
    }
}
```

# CNV-Seq
Now run the main perl script: 

`cnv-seq.pl --ref ref.hits --test sample.qfilt.hits --genome-size 23542271`

Options:
```
	--test = test.hits.file
	--ref = ref.hits.file

	--log2-threshold = number
	  (default=0.6)
	--p-value = number
	  (default=0.001)

	--bigger-window = number
	  (default=1.5)
	--window-size
	  (default: determined by log2 and p-value;
	   if set, then log2 and p-value are not used)

	--genome = (human, chicken, chrom1, autosome, sex, chromX, chromY)
	  (higher priority than --genome-size)
	--genome-size = number
	  (no use when --genome is avaliable)

	--annotate
	--no-annotate
	  (default: do annotation, ie --annotate)
	--minimum-windows-required = number 
	  (default=4; only for annotation of CNV)
	--global-normalization
	  (if used, normalization on whole genome,
	  instead of oneach chromosome)

	--Rexe = path to your R program
	  (default: R)

	--help
	--debug
```

The most interesting parameter that can be tweaked is the `--log2-threshold`. The default is set to 0.6. (Not clear how changin this affects results though...)

# Output

This produces two files `sample-vs-reference.cnv` and `sample-vs-reference.count`

`sample-vs-reference.count` shows the raw count data for each CNV. 

| chromosome | start | end | test | ref |
|:---:|:---:|:---:|:---:|:---:|
| X | 1 | 363 | 70 | 124 |
| X | 183 | 545 | 82 | 123 |
| X | 365 | 727 | 90 | 115 |

`sample-vs-reference.cnv` contains the stats. 

| chromosome | start | end | test | ref | position | log2 | p.value | cnv | cnv.size | cnv.log2 | cnv.p.value |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| X | 1 | 363 | 70 | 124 | 182 | -0.479319752689881 | 0.0462467378993667 | 0 | NA | NA | NA |
| X | 183 | 545 | 82 | 123 | 364 | -0.239368959969129 | 0.194296291525077 | 0 | NA | NA | NA |
| X | 365 | 727 | 90 | 115 | 546 | -0.00804341386267303 | 0.488268352520283 | 0 | NA | NA | NA |


## Selecting for high and low coverage regions

In order to filter out CNVs of interest run `top_change.pl`, which will output IGV compatible annotation tracks (`_topchange_up.gff3` and `_topchange_down.gff3`) of CNVs above and below a user specified log2 threhold e.g. abs(log2) > `1` (linear FC 2).
These can then be used for plotting or visualising in IGV.

## Plotting

Initially, plot all data for all chromosomes:
```{R}
library(cnv)
Hum1_Hum3 <- read.delim("Hum1.hits-vs-Hum3.hits.log2-0.8.pvalue-0.001.minw-4.cnv")
plot.cnv(Hum1_Hum3, ylim = c(-8,8))
```

Can also plot per chromosome: 

```{R}
plot.cnv(Hum1_Hum3, ylim = c(-8,8), chromosome = "X")
```

Or check out CNVs identified, and plot an interesting region:

```{R}
cnv.print(Hum1_Hum3)
# Select CNV #43, and plot surrounding region +/- 10kb
plot.cnv(Hum1_Hum3, CNV=43, upstream=1e+4, downstream=1e+4)
```


# To do

Lots more plotting features to expore. The following needs tweaking. 
Ideally, plot chromosomes in a grid, one per row on one page.

```{R}
for (chrom in c("X","Y")){
	plot.cnv.chr <- function (data, chromosome=chrom,
	                      from = NA, to = NA, title = paste("full coverage of",chrom, sep="")
	                      ylim = c(-4, 4), glim = c(NA, NA),
	                      xlabel = "Position (bp)")
	plot.cnv.chr
	ggsave(file = paste("HUM-1",chrom,".pdf",sep="_"))
}
```
						  
Should also look into filtering output files to select regions above certain thresholds. E.g.:
* CNVs above/below size threshold
* CNVs supported by > x reads (av coverage depth?)