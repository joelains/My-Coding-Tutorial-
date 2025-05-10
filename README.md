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
  Deblur uses sequence error profiles to associate wrong sequences with true biological sequences which results high quality sequence variant data. First it will go through quality filtering process.
 
  qiime quality-filter q-score \
  --i-demux single-end-demux.qza \
  --o-filtered-sequences demux-filtered.qza \
  --o-filter-stats demux-filter-stats.qza
  
  Second step is to denoise and use the qiime deblur denoise-16S method which requires trim length. However since each of sequences in the samples are 301 we must keep it at 301 because all of the base pairs important for the run. -p jobs-to-start 6 , reqired so the denoise process can be quicker than normal and save time for analysis. 
  
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
## Step 3: Feature Table and Summaries 
After the denoising step I'm going 
  
