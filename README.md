# My-Coding-Tutorial-
How I coded
## Requirements
- Linux (Ubuntu, Debian, or other)
- Conda (Miniconda or Anaconda)
- QIIME 2 (2024.2 release)
- ## Step 1: Import Data
  Because each of samples is one patient i must use a manfiest file in order for qiime2 to recongize each of the reads seperately. I must make a TSV table with the sample ID's, path to the directory in which qiime2 is being ran and the direction of the sequences (forward or revers) which is important for the SingleEndFastqManifestPhred33V2 command. I made a folder in the absolute path with those samples. 

qiime tools import \
  --type 'SampleData[SequencesWithQuality]' \
  --input-path GERD.TSV \
  --output-path single-end-demux.qza \
  --input-format SingleEndFastqManifestPhred33V2
  
  ## Step 2: Denoising with Deblur 
  Deblur uses sequence error profiles to associate wrong sequences with true biological sequences, resulting in high-quality sequence variant data. First it will go through quality filtering process.
 
  qiime quality-filter q-score \
  --i-demux single-end-demux.qza \
  --o-filtered-sequences demux-filtered.qza \
  --o-filter-stats demux-filter-stats.qza
  
  The second step is to denoise and use the Qiime deblur denoise-16S method, which requires a trim length. However, since each of the sequences in the samples is 301 we must keep it at 301 because all of the base pairs are important for the run. -p jobs-to-start 6 , reqired so the denoise process can be quicker than normal and save time for analysis. 
  
  qiime deblur denoise-16S \
  --i-demultiplexed-seqs demux-filtered.qza \
  --p-trim-length 301 \
  --p-sample-stats \
  --p-jobs-to-start 6 \
  --o-representative-sequences rep-seqs-deblur.qza \
  --o-table table-deblur.qza \
  --o-stats deblur-stats.qza

  To continue using the artifacts for the feature table i ran this. 
  mv rep-seqs-deblur.qza rep-seqs.qza
  mv table-deblur.qza table.qza
## Step 3: Feature Table and Merging Data
After the denoising step, I will create visual summaries of the data. 

qiime feature-table summarize \
  --i-table table.qza \
  --m-sample-metadata-file sample-metadata.tsv \
  --o-visualization table.qzv
qiime feature-table tabulate-seqs \
  --i-data rep-seqs.qza \
  --o-visualization rep-seqs.qzv

  Since I got the data from different datasets, I must merge the feature tables and sequences to analyze their differences and relationships. Also, merged the metadata, in my case i just combined the most important factors and made sure i had sample ID and study.
  

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
  
  ##Step 4: Generate a tree for phylogenetic diversity analyses 
  A rooted phylogenetic tree is required for diversity analysis. I'll be using a Mafft program to peform multiple sequnce alignment from our sequences from each of the samples in the merged-rep-seqs
  
  qiime phylogeny align-to-tree-mafft-fasttree \
  --i-sequences merged-rep-seqs.qza \
  --o-alignment aligned-rep-seqs.qza \
  --o-masked-alignment masked-aligned-rep-seqs.qza \
  --o-tree unrooted-tree.qza \
  --o-rooted-tree rooted-tree.qza

  ##Step 5: Alpha/Beta Diveristy Analysis
  Before i run the analysis. I must check a few things i have to look at the distriubtion of the frequncy per samples there seems to be a great amount of cluster from around 0-16649 frequency per sample. So i adjust the frequency. 
  
  qiime feature-table filter-samples \
  --i-table merged-table.qza \
  --p-max-frequency 16649 \
  --o-filtered-table table-no-high.qza

  Then we must set sampling dpeth to a parameter 1.... This was chose because the sample number of sequences in the merged metadata is closest to the rest of higher sequence counts so the ones with least will be dropped from the analysis. 

qiime diversity core-metrics-phylogenetic \
  --i-phylogeny rooted-tree.qza \
  --i-table table-no-high.qza \
  --p-sampling-depth 1256 \
  --m-metadata-file MergedMD.tsv \
  --output-dir diversity-core-metrics-phylogenetic

  After commuting the diverstiy metrics we can explor the microbiom composition of the samples in the merged dataset. I'll focus on alpha diversity and by doing the faith PD and eveness
  
  qiime diversity alpha-group-significance \
  --i-alpha-diversity diversity-core-metrics-phylogenetic/faith_pd_vector.qza \
  --m-metadata-file MergedMD.tsv \
  --o-visualization faith-pd-group-significance.qzv
qiime diversity alpha-group-significance \
  --i-alpha-diversity diversity-core-metrics-phylogenetic/evenness_vector.qza \
  --m-metadata-file MergedMD.tsv \
  --o-visualization evenness-group-significance.qzv
  
