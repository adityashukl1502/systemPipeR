---
title: 6. Peak calling with MACS2
last_updated: Thu Nov 21 15:49:32 2019
sidebar: mydoc_sidebar
permalink: mydoc_systemPipeChIPseq_06.html
---

## Merge BAM files of replicates prior to peak calling

Merging BAM files of technical and/or biological replicates can improve
the sensitivity of the peak calling by increasing the depth of read
coverage. The `mergeBamByFactor` function merges BAM files based on grouping information
specified by a `factor`, here the `Factor` column of the imported targets file. It 
also returns an updated `SYSargs2` object containing the paths to the
merged BAM files as well as to any unmerged files without replicates.
This step can be skipped if merging of BAM files is not desired.


```r
dir_path <- system.file("extdata/cwl/mergeBamByFactor", package = "systemPipeR")
args <- loadWF(targets = "targets_bam.txt", wf_file = "merge-bam.cwl", 
    input_file = "merge-bam.yml", dir_path = dir_path)
args <- renderWF(args, inputvars = c(FileName = "_BAM_PATH_", 
    SampleName = "_SampleName_"))

args_merge <- mergeBamByFactor(args = args, overwrite = TRUE)
writeTargetsout(x = args_merge, file = "targets_mergeBamByFactor.txt", 
    step = 1, new_col = "FileName", new_col_output_index = 1, 
    overwrite = TRUE)
```


```r
# Skip if a module system is not used
module("list")
module("unload", "miniconda2")
module("load", "python/2.7.14")  # Make sure to set up your enviroment variable for MACS2 
```

## Peak calling without input/reference sample

MACS2 can perform peak calling on ChIP-Seq data with and without input
samples (Zhang et al., 2008). The following performs peak calling without
input on all samples specified in the corresponding `args` object. Note, due to
the small size of the sample data, MACS2 needs to be run here with the
`nomodel` setting. For real data sets, users want to remove this parameter 
in the corresponding `*.param` file(s).


```r
dir_path <- system.file("extdata/cwl/MACS2/MACS2-noinput/", package = "systemPipeR")
args <- loadWF(targets = "targets_mergeBamByFactor.txt", wf_file = "macs2.cwl", 
    input_file = "macs2.yml", dir_path = dir_path)
args <- renderWF(args, inputvars = c(FileName = "_FASTQ_PATH1_", 
    SampleName = "_SampleName_"))

runCommandline(args, make_bam = FALSE, force = T)
outpaths <- subsetWF(args, slot = "output", subset = 1, index = 1)
file.exists(outpaths)
writeTargetsout(x = args, file = "targets_macs.txt", step = 1, 
    new_col = "FileName", new_col_output_index = 1, overwrite = TRUE)
```

## Peak calling with input/reference sample

To perform peak calling with input samples, they can be most
conveniently specified in the `SampleReference` column of the initial
`targets` file. The `writeTargetsRef` function uses this information to create a `targets` 
file intermediate for running MACS2 with the corresponding input samples.


```r
writeTargetsRef(infile = "targets_mergeBamByFactor.txt", outfile = "targets_bam_ref.txt", 
    silent = FALSE, overwrite = TRUE)
dir_path <- system.file("extdata/cwl/MACS2/MACS2-input/", package = "systemPipeR")
args_input <- loadWF(targets = "targets_bam_ref.txt", wf_file = "macs2-input.cwl", 
    input_file = "macs2.yml", dir_path = dir_path)
args_input <- renderWF(args_input, inputvars = c(FileName1 = "_FASTQ_PATH1_", 
    FileName2 = "_FASTQ_PATH2_", SampleName = "_SampleName_"))
cmdlist(args_input)[1]
## Run
args_input <- runCommandline(args_input, make_bam = FALSE, force = T)
outpaths_input <- subsetWF(args_input, slot = "output", subset = 1, 
    index = 1)
file.exists(outpaths_input)
writeTargetsout(x = args_input, file = "targets_macs_input.txt", 
    step = 1, new_col = "FileName", new_col_output_index = 1, 
    overwrite = TRUE)
```

The peak calling results from MACS2 are written for each sample to
separate files in the `results` directory. They are named after the corresponding
files with extensions used by MACS2.

## Identify consensus peaks

The following example shows how one can identify consensus preaks among two peak sets sharing either a minimum absolute overlap and/or minimum relative overlap using the `subsetByOverlaps` or `olRanges` functions, respectively. Note, the latter is a custom function imported below by sourcing it.


```r
# source('http://faculty.ucr.edu/~tgirke/Documents/R_BioCond/My_R_Scripts/rangeoverlapper.R')
outpaths <- subsetWF(args, slot = "output", subset = 1, index = 1)  ## escolher um dos outputs index
peak_M1A <- outpaths["M1A"]
peak_M1A <- as(read.delim(peak_M1A, comment = "#")[, 1:3], "GRanges")
peak_A1A <- outpaths["A1A"]
peak_A1A <- as(read.delim(peak_A1A, comment = "#")[, 1:3], "GRanges")
(myol1 <- subsetByOverlaps(peak_M1A, peak_A1A, minoverlap = 1))
# Returns any overlap
myol2 <- olRanges(query = peak_M1A, subject = peak_A1A, output = "gr")
# Returns any overlap with OL length information
myol2[values(myol2)["OLpercQ"][, 1] >= 50]
# Returns only query peaks with a minimum overlap of 50%
```

<br><br><center><a href="mydoc_systemPipeChIPseq_05.html"><img src="images/left_arrow.png" alt="Previous page."></a>Previous Page &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; Next Page
<a href="mydoc_systemPipeChIPseq_07.html"><img src="images/right_arrow.png" alt="Next page."></a></center>