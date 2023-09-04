# Wine Metagenomics
Practice project for metagenomic analysis of fungal community composition of fermenting wine must using Kraken2.

### Download wine fermentation metagenome data
Download fungal amplicon metagenomic sequencing data representing four time points of a Chardonnay must fermentation.
```
bash ena-file-download-selected-files-20230904-1049.sh
```

## Kraken2

Following tutorial in [2].

### Install Kraken2 and related tools
```
conda create -n kraken_env kraken2 bracken krakentools
conda activate kraken_env
```

### Shared Kraken databases
```
/mnt/shared/apps/databases/kraken/
/mnt/shared/apps/databases/krakenuniq/custom_kraken2_rmacleod/kraken2/
```

### Download and build fungal db
```
srsh --mem=16G --cpus-per-task=10

# First, download the NCBI taxonomy
kraken2-build --db krakendb --download-taxonomy

# Second, download one or more reference libraries
kraken2-build --db krakendb --download-library fungi

# Finally, build the Kraken 2 database and generate the Bracken database files
kraken2-build --db krakendb --build --threads 8
bracken-build -d krakendb -t 8 -k 35 -l 100

```

### Classify microbiome samples using Kraken2
```
srsh --mem=16G --cpus-per-task=10

for file in data/*_1.fastq.gz;
    do short=$(basename $file | sed s/"_1.fastq.gz"//g)
    kraken2 --db db/krakendb --threads 8 --report kraken2/"$short".k2report --report-minimizer-data \
        --minimum-hit-groups 3 $file data/"$short"_2.fastq.gz > kraken2/"$short".kraken2
    done
```

### Run bracken for abundance estimation of microbiome samples
Set the read length as the average read length in the dataset: -r 100. We set the level for abundance estimation to species (with -l S), and with the -t 10 we require ten reads before abundance estimation to perform re-estimation. This effectively removes any species with fewer than ten reads, thereby removing some noise from low-abundance species.

```
for file in kraken2/*.k2report;
    do short=$(basename $file | sed s/".k2report"//g)
    bracken -d db/krakendb -i $file -r 100 -l S -t 10 \
        -o bracken/$short.bracken -w bracken/$short.breport
    done
```

### Calculate α-diversity
Using KrakenTools alpha_diversity.py script, calculate Berger Parker’s33 (BP), Fisher’s34 (F), Simpson’s35 (Si), inverse Simpson’s (ISi)35 and Shannon’s36 (Sh) α-diversity for each sample.

```
for file in bracken/*.bracken;
    do short=$(basename $file | sed s/".braken"//g)
    alpha_diversity.py -f $file -a BP >> bracken/$short.alpha
    alpha_diversity.py -f $file -a F >> bracken/$short.alpha
    alpha_diversity.py -f $file -a Si >> bracken/$short.alpha
    alpha_diversity.py -f $file -a ISi >> bracken/$short.alpha
    alpha_diversity.py -f $file -a Sh >> bracken/$short.alpha
    done
```

### Calculate β-diversity
β-Diversity is useful when trying to examine the change in species diversity between two or more samples. Compute the Bray–Curtis dissimilarity matrix, which will contain the pairwise dissimilarities, an output of 0 means the two samples are exactly the same and 1 means they are maximally divergent.

```
beta_diversity.py -i \
    bracken/SRR10290077.bracken \
    bracken/SRR10290078.bracken \
    bracken/SRR10290085.bracken \
    bracken/SRR10290086.bracken \
    --type bracken > bracken/beta.diversity
```

### Generate Krona plots
Krona plots are multilayered pie charts frequently used in metagenomic visualization for viewing data in a phylogenetic hierarchy. 

```
for file in bracken/*.breport;
    do short=$(basename $file | sed s/".breport"//g)
    kreport2krona.py -r $file -o krona/$short.b.krona.txt --no-intermediate-ranks
    ktImportText krona/$short.b.krona.txt -o krona/$short.krona.html
    done
```

### Generate Pavian plots using the Shiny app
Open the Pavian Shiny app via https://fbreitwieser.shinyapps.io/pavian/. Upload the .breport files and click ‘Sample’ to see the hierarchical classification visualization results.


# References
[1] https://www.nature.com/articles/s41598-023-40799-x \
[2] https://www.nature.com/articles/s41596-022-00738-y \
[3] https://genomics.sschmeier.com/ngs-taxonomic-investigation/index.html \
