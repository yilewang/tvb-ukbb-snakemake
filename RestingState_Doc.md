# Steps of resting state fMRI preprocessing in tvb-ukbb pipeline

This doc is to summarize the resting state fmri preprocessing in the tvb-ukbb pipeline. The resting state pipeline will use the outputs from structural pipeline (T1, brain-extracted T1 (after BET), linear and non-linear warp pf T1 to MNI space and binary mask of white matter in T1). The flow chart of the rs-fmri preprocessing is here:

![image](https://user-images.githubusercontent.com/37648360/157544150-6e5ceb0d-90f2-4e32-908c-f36b56fc65bf.png)

![image](https://user-images.githubusercontent.com/37648360/157547046-4d802439-6e39-4777-b9fd-5e9c0985a109.png)

## 1. post structural

Post structural step aims to collect the necessary files from T1w analyses. Bascially, it will need `T1.nii.gz`,`T1_brain.nii.gz`, T1 to MNI152 results `T1_brain2MNI152_T1_2mm_brain_warp.nii.gz` files and partial volume image (PVE) file (to create a mask called `WM_seg.nii.gz`). 

```bash
direc=$PWD/$1

linkDir=$direc/fMRI/symlink

mkdir $linkDir

ln -s ${direc}/T1/T1.nii.gz $linkDir/T1.nii.gz
ln -s ${direc}/T1/T1_brain.nii.gz $linkDir/T1_brain.nii.gz
ln -s ${direc}/T1/transforms/T1_to_MNI_warp.nii.gz $linkDir/T1_brain2MNI152_T1_2mm_brain_warp.nii.gz
ln -s ${direc}/T1/transforms/T1_to_MNI_linear.mat $linkDir/T1_brain2MNI152_T1_2mm_brain.mat

$FSLDIR/bin/fslmaths $direc/T1/T1_fast/T1_brain_pve_2.nii.gz -thr 0.5 -bin $linkDir/T1_brain_wmseg.nii.gz

```


## 2. rs-fMRI FEAT/Melodic

FEAT is a software tool for high quality model-based FMRI data analysis.

Melodic is Multivariate Exploratory Linear Optimized Decomposition into Independent Components. 

## 3. FIX

## 4. functional connectivity
