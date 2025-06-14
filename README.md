# My-Coding-Tutorial-
How I coded
## Requirements
- Linux (Ubuntu, Debian, or other)
- Conda (Miniconda or Anaconda)
- QIIME 2 (2025.04 release)
  
# Step 1: Import Data

  Since each sample represents one patient, I needed to use a manifest file so that QIIME 2 could recognize each read separately. I created a tab-separated values (TSV) file with the sample IDs, absolute file paths to the fastq files, and the direction of the reads (forward or reverse). This information is required for the SingleEndFastqManifestPhred33V2 import format. I placed the fastq files in a folder located at the path where QIIME 2 is being run.

qiime tools import \
  --type 'SampleData[SequencesWithQuality]' \
  --input-path GERD.TSV \
  --output-path single-end-demux.qza \
  --input-format SingleEndFastqManifestPhred33V2
  
# Step 2: Denoising with Deblur 

Deblur uses sequence error profiles to distinguish sequencing errors from true biological sequences, producing high-quality amplicon sequence variants (ASVs). The first step in the pipeline is quality filtering:

  qiime quality-filter q-score \
  --i-demux single-end-demux.qza \
  --o-filtered-sequences demux-filtered.qza \
  --o-filter-stats demux-filter-stats.qza
  
Next, I denoised the sequences using the deblur denoise-16S method. I set the trim length to 301 bp to retain the full sequence length, which is necessary because all base pairs are important for my analysis. I also specified six jobs to speed up the denoising process.
  
  qiime deblur denoise-16S \
  --i-demultiplexed-seqs demux-filtered.qza \
  --p-trim-length 301 \
  --p-sample-stats \
  --p-jobs-to-start 6 \
  --o-representative-sequences rep-seqs-deblur.qza \
  --o-table table-deblur.qza \
  --o-stats deblur-stats.qza

To continue using the outputs, I renamed the artifacts: 
  mv rep-seqs-deblur.qza rep-seqs.qza
  mv table-deblur.qza table.qza
# Step 3: Feature Table and Merging Data
After denoising, I created visual summaries of the feature table and representative sequences:

qiime feature-table summarize \
  --i-table table.qza \
  --m-sample-metadata-file sample-metadata.tsv \
  --o-visualization table.qzv
qiime feature-table tabulate-seqs \
  --i-data rep-seqs.qza \
  --o-visualization rep-seqs.qzv

Because I used data from multiple datasets, I needed to merge the feature tables and representative sequences to analyze differences and relationships across studies. I also merged metadata files, combining the most important fields and ensuring each sample had a unique ID and study label.  

  qiime feature-table merge \
  --i-tables GERDtable.qza \
  --i-tables Crohnstable.qza \
  --i-talbes HealthyEtable.qza \
  --o-merged-table merged-table.qza

  qiime feature-table merge-seqs \
  --i-data GERDrep-seqs.qza \
  --i-data HealthyErep-seqs.qza \
  --i-data Crohnsrep-seqs.qza \
  --o-merged-data merged-rep-seqs.qza
  
# Step 4: Generate a tree for phylogenetic diversity analyses 
To perform phylogenetic diversity analysis, a rooted tree is required. I used the align-to-tree-mafft-fasttree pipeline to align sequences and generate both unrooted and rooted trees.

  qiime phylogeny align-to-tree-mafft-fasttree \
  --i-sequences merged-rep-seqs.qza \
  --o-alignment aligned-rep-seqs.qza \
  --o-masked-alignment masked-aligned-rep-seqs.qza \
  --o-tree unrooted-tree.qza \
  --o-rooted-tree rooted-tree.qza

# Step 5: Alpha/Beta Diversity Analysis
Before running diversity metrics, I checked the distribution of read counts per sample. Based on this, I filtered out samples with extremely high read counts (above 16,649):

  qiime feature-table filter-samples \
  --i-table merged-table.qza \
  --p-max-frequency 16649 \
  --o-filtered-table table-no-high.qza

Next, I set a sampling depth for rarefaction. I chose 1256 because it balanced the inclusion of most samples while avoiding overly shallow sequences.

qiime diversity core-metrics-phylogenetic \
  --i-phylogeny rooted-tree.qza \
  --i-table table-no-high.qza \
  --p-sampling-depth 1256 \
  --m-metadata-file MergedMD.tsv \
  --output-dir diversity-core-metrics-phylogenetic

Finally, I explored microbial diversity by focusing on alpha diversity metrics like Faith's Phylogenetic Diversity and Pielou's Evenness:

  qiime diversity alpha-group-significance \
  --i-alpha-diversity diversity-core-metrics-phylogenetic/faith_pd_vector.qza \
  --m-metadata-file MergedMD.tsv \
  --o-visualization faith-pd-group-significance.qzv
qiime diversity alpha-group-significance \
  --i-alpha-diversity diversity-core-metrics-phylogenetic/evenness_vector.qza \
  --m-metadata-file MergedMD.tsv \
  --o-visualization evenness-group-significance.qzv

Next, we'll explore the beta group significance using PERMANOVA, which tests the distance between samples within a group, such as different disease states (Crohn's v GERD). We will run the command p-pairwise parameter to determine the groups that are statistically different from each other.

qiime diversity beta-group-significance \
  --i-distance-matrix diversity-core-metrics-phylogenetic/unweighted_unifrac_distance_matrix.qza \
  --m-metadata-file MergedMD.tsv \
  --m-metadata-column group \
  --p-pairwise \
  --o-visualization unweighted-unifrac-subject-group-significance.qzv

  To better visualize the microbial community composition, we can use the emperor plot to explore principal coordinates plots. We will use the unweighted UniFrac to generate emperor plots. 

qiime emperor plot \
  --i-pcoa diversity-core-metrics-phylogenetic/unweighted_unifrac_pcoa_results.qza \
  --m-metadata-file MergedMD.tsv \
  --o-visualization unweighted-unifrac-emperor.qzv
  
# Step 6: Assigning Taxonomy 
Now we'll see the samples' taxonomic composition and relate it back to the metadata. Sadly, due a limited RAM memory, i used a pre-trained Naive Bayes by Greengenes 13_8 which have been trimmed to inlclude the v4 region (my sequences were v3-v4). Fortunately results were reliable. 
wget -O 'gg-13-8-99-515-806-nb-classifier.qza' \
  'https://moving-pictures-tutorial.readthedocs.io/en/latest/data/moving-pictures/gg-13-8-99-515-806-nb-classifier.qza'

  qiime feature-classifier classify-sklearn \
  --i-classifier gg-13-8-99-515-806-nb-classifier.qza \
  --i-reads merged_seqs.qza \
  --o-classification taxonomy.qza
qiime metadata tabulate \
  --m-input-file taxonomy.qza \
  --o-visualization taxonomy.qzv

Next, we can view the composition of our samples by generating bar plots. 

qiime taxa barplot \
  --i-table table-no-high.qza \
  --i-taxonomy taxonomy.qza \
  --m-metadata-file MergedMD.tsv \
  --o-visualization taxa-bar-plots.qzv
  
# Step 7: Differential Abundance Test
Using ANCOM-BC to test for abundant features across the groups while implementing a bias correction. For this research, we are only focusing on Crohn's vs GERD samples to view the relationships of the two different microbiomes (gut v esophagus).

qiime taxa collapse \
  --i-table table-no-high.qza \
  --i-taxonomy taxonomy.qza \
  --p-level 6 \
  --o-collapsed-table genus-table-crohns-gerd.qza

qiime metadata filter
--m-metadata-file MergedMD.tsv \
--p-where "[group] IN ('crohns', 'gerd')" \
--o-filtered-table MergedMD.tsv 

qiime composition ancombc \
  --i-table genus-table-crohns-gerd.qza \
  --m-metadata-file MergedMD.tsv \
  --p-formula group \
  --o-differentials l6-ancombc-subject.qza

qiime composition da-barplot \
  --i-data l6-ancombc-subject.qza \
  --p-significance-threshold 0.001 \
  --o-visualization l6-da-barplot-subject.qzv

  # END

