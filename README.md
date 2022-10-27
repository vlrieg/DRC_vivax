This repository contains code and steps used for investigating *Plasmodium vivax* evolutionary history in central Africa using global *P. vivax* whole genome sequencing data. Results of this work will be available on bioRxiv soon. <mark>To do: add bioRxiv DOI</mark>

Below is an outline of what tools and commands were used for this work.

<mark>To do:</mark>
- Include which figure corresponds to each directory/analysis
- Double check for any remaining hard coded paths/arguments

# sample_info directory
<mark>To Do:</mark>
- sample table (accession, species, region, country, MOI)
- Description of what each file/script does
  - Rmd document for pulling out different subregions

Determining MOI

1.  generate two vcfs: one polyclone with a max of 3 clones, and one a gvcf which indicates the level of coverage at each base.  The commands to generate these are in `make-fastas-6.sh`
```
octopus -I \${DEDUP_BAM} -R ${REF} -T LT635626 -o api.poly3.vcf.gz --annotations AD -C polyclone  --max-clones 3 --threads 16 --sequence-error-model PCR
octopus -I \${DEDUP_BAM} -R ${REF} -T LT635626 -o api.g.vcf.gz --annotations AD --refcall POSITIONAL --threads 16 --sequence-error-model PCR
```
2. `check-accessions.py` scans the genomes2 directory that contains directories/files like ERR12355/api.poly3.vcf.gz .  It produced the simple text file genomes2/mono-0.9.txt  with lines like
```
ERR773745       OK
ERR773746       PolyClonal
ERR773747       NoGVCF
ERR773748       OK
```
The 0.9 indicates that a site is considered homozygous if the major allele frequency is 0.9. Run it like `check-accesions.py --cutoff 0.9 --allowed-het-sites=1 > mono-0.9.txt`

Create the list of accessions with `grep OK mono-0.9.txt | cut -f1 > mono-0.9-accs.txt`

Accessions with no MOI data were included by default.

# genome_processing directory

<mark>To Do:</mark>

Description of what each file/script does
- get-haploid-gvcf.sh
  - arguments:
  - example file format
  - adjusting array size notes

Notes:
- to use this script, change "/path/to/picard.jar" and "/path/to/GenomeAnalysisTK.jar" to appropriate paths for your machine
- also update/remove java module load commands for your machine as needed

add-sample-name.sh

joint call step

keep only chromosomes (no contigs) and remove masked regions
& also keep snps only script

# Analysis directory

## Figure 1C: Admixture

Starting with SNPs-only VCF file (chromosomes only & masked regions removed)

Downsample loci
```
sort -R min_filt_no-singletons.recode.pruned.genotypes.bim | head -n 100000 | awk '{print $2}' > random100k.snps
#https://www.biostars.org/p/16038/#16085

# change format
# old: chr:pos
# now: chr	pos
sed "s/\:/\t/g" random100k.snps > random100k.snps.txt

# extract random positions
bcftools view -R random100k.snps.txt min_filt_no-singletons.recode.vcf.gz > random100k_min_filt_no-singletons.recode.vcf

# SORT positions with vcftools 'vcf-sort' tool
cat random100k_min_filt_no-singletons.recode.vcf | vcf-sort > random100k-SORTED_min_filt_no-singletons.recode.vcf
```

Convert PvP01 chromosome names to integers
`for i in $(<replace-chr-w-ints_sed-arguments.txt ) ; do sed -i ${i} chr-as-int_global_vivax.vcf ; done`

https://github.com/stevemussmann/admixturePipeline
in script [admixture.py](https://github.com/stevemussmann/admixturePipeline/blob/master/admixture.py), update the command string to indicate haploid mode
```
#### HAPLOID MODE
command_string = "admixture" + " -s " + str(np.random.randint(1000000)) + " --cv=" + str(self.cv) + " " + self.prefix + ".ped " + str(i) + " " + haploid_str
self.run_program(command_string,i,j)
```

[Pong](https://github.com/ramachandran-lab/pong)

Prepare information files for generating Admixture visualization
```
# File map
for i in *.Q ; do j=$i ; i=${i##*.pruned.genotypes.} ; echo -e "k-${i%%.Q}\t${i%%_[0-9]*.Q}\t${j}" ; done > filemap.txt

# Pop order
awk '{print $2}' popmap.txt |sort|uniq > pop_order.txt
# then manually arrange the order in this file to represent left to right on map	

# Mapping individuals to populations
awk '{print $2}' popmap.txt > ind2pop.txt


#Run Pong
`pong -m filemap.txt -i ind2pop.txt -n pop_order_revised.txt -v`
```

To generate Cross Validation Error box plots:
```
for i in *.stdout ; do grep -h "CV error" ${i} >> overall_cv_summary.txt ; done
awk '{print $3"\t"$4}' overall_cv_summary.txt | tr -d '():K=' >> cv_summary_table.txt
```

And visualize with `CrossValidationError_boxplots.Rmd`


## Figure 1A: PCA
Using Plink files generated by AdmixturePipeline

`plink --bfile global_population.genotypes --pca # genotype files from admixturePipeline step previously completed`

- Rmd document to do visualizations


## Figure 1B: Phylogenetic trees
Starting with gvcf file (NOT SNPs only file)

1. Convert
[vcf2phylip](https://github.com/joanam/scripts/blob/master/vcf2phylip.py)

[remove invariant sites](https://github.com/btmartin721/raxml_ascbias)

- run iqtree script



## Table 1: Summary Statistics

<mark>commands to pull out biallelic SNPs for each population</mark>


`for i in *_minfilt_global-subsample.g_masked-rm.recode.chroms-only_chr10-subtel-rm.recode_biallelic_snps_only_min-ac1.vcf.gz ; do ./segregating-sites-from-vcf.py --vcf ${i} ; done`

Create window bedfile

`bedtools makewindows -b PVP01.chroms.bed -w 1000 > PVP01.chroms_1kb-windows.bed`

For each population:

`./run-pixy-bedfile.sh


Visualize using `visualize-private-alleles.Rmd`

## PvDBP Duplication/CNV
- <mark>To Do: include bedtools commands and Rmd for visualizations</mark>


Fastq > align to PvP01 reference > remove optical duplicates > ${BAM}.dedup.bam

Pull out region surrounding PvDBP

- `samtools view -bh file.dedup.bam "LT635617:970474-998688" > DBP-EXTRA-extended_file.dedup.bam`

Sort the BAM

- `samtools sort -o file.dedup.sorted.bam file.dedup.bam`

Use [bedtools genomecov](https://bedtools.readthedocs.io/en/latest/content/tools/genomecov.html) to find the per-site read depth

- `bedtools genomecov -d -ibam file.dedup.sorted.bam > file.dedup.sorted.persitedepth.bedgraph`

Visualize with `coverage-plots.Rmd`