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

Before entering the rs-fmri analysis, we also need to create a `fmri.fsf` file by `fmri prepare design file generator`:

```bash
# append the following to the rfMRI.fsf design file
echo "set fmri(outputdir)     \""$nameDir"/fMRI/rfMRI.ica\"" >> $1/fMRI/rfMRI.fsf
echo "set fmri(regunwarp_yn) 0" >> $1/fMRI/rfMRI.fsf
echo "set feat_files(1)       \""$nameDir"/fMRI/rfMRI.nii.gz\"" >> $1/fMRI/rfMRI.fsf
echo "set alt_ex_func(1)      \""$nameDir"/fMRI/rfMRI_SBREF.nii.gz\"" >> $1/fMRI/rfMRI.fsf
#echo "set unwarp_files(1)     \""$nameDir"/fieldmap/fieldmap_fout_to_T1_brain_rad.nii.gz\"" >> $1/fMRI/rfMRI.fsf
#echo "set unwarp_files_mag(1) \""$nameDir"/fMRI/symlink/T1_brain.nii.gz\"" >> $1/fMRI/rfMRI.fsf
echo "set unwarp_files(1) 0" >> $1/fMRI/rfMRI.fsf
echo "set unwarp_files_mag(1) 0" >> $1/fMRI/rfMRI.fsf
echo "set highres_files(1)    \""$nameDir"/fMRI/symlink/T1_brain.nii.gz\"" >> $1/fMRI/rfMRI.fsf
#echo "set fmri(gdc)           \""$nameDir"/fMRI/rfMRI_SBREF_ud_warp.nii.gz\"" >> $1/fMRI/rfMRI.fsf
echo "set fmri(unwarp_dir) 0" >> $1/fMRI/rfMRI.fsf

echo "set fmri(tr) $fmriTR" >> $1/fMRI/rfMRI.fsf
echo "set fmri(npts) $fmriNumVol"  >> $1/fMRI/rfMRI.fsf
echo "set fmri(dwell) $fmriDwell" >> $1/fMRI/rfMRI.fsf
echo "set fmri(te) $fmriTE" >> $1/fMRI/rfMRI.fsf
echo "set fmri(smooth) 5" >> $1/fMRI/rfMRI.fsf
echo "set fmri(ndelete) 4" >> $1/fMRI/rfMRI.fsf

. $BB_BIN_DIR/bb_pipeline_tools/bb_set_footer 
```




## 2. rs-fMRI FEAT/Melodic

FEAT is a software tool for high quality model-based FMRI data analysis. We didn't use this tool in the pipeline because it is for General linear modeling in Task-fmri.

Melodic is Multivariate Exploratory Linear Optimized Decomposition into Independent Components. It is for resting-state fmri. The 

![image](https://user-images.githubusercontent.com/37648360/159042722-b4044789-0d1e-414f-86ac-f56f2462a1b4.png)

To make sure the registration across different modalities is good, the Boundary-Based Registration (BBR) will be applied first in the fMRI data. 
![image](https://user-images.githubusercontent.com/37648360/159046455-69ac7a02-4513-4526-9771-06cdc47f243c.png)


## 3. FIX

FIX is a supervised learning tool to label the signal or noise components for ICA results. It required a training file at the first for at least 20 people. The accuracy of the FIX can reach 99% in high quality data.

```bash

[ "$1" = "" ] && exit

if [ -d $1/fMRI/rfMRI.ica ] ; then 

    cd $1/fMRI/rfMRI.ica

    ${FSLDIR}/bin/imcp ${origDir}/$1/T1/T1_fast/T1_brain_pveseg reg/highres_pveseg
    # ${FSLDIR}/bin/imcp $1/T1/T1_fast/T1_brain_pveseg reg/highres_pveseg
    ${FSLDIR}/bin/invwarp --ref=reg/example_func -w reg/example_func2standard_warp -o reg/standard2example_func_warp
    ${FSL_FIXDIR}/fix . ${FSL_FIXDIR}/training_files/${TRAINING_FILE} 30 -m -h 100

    mkdir -p reg_standard
    ${FSLDIR}/bin/applywarp --ref=reg/standard --in=filtered_func_data_clean --out=reg_standard/filtered_func_data_clean --warp=reg/example_func2standard_warp --interp=spline

    ${FSLDIR}/bin/fslmaths reg_standard/filtered_func_data_clean -mas $templ/MNI152_T1_2mm_brain_mask_bin reg_standard/filtered_func_data_clean
    ${FSLDIR}/bin/fslmaths reg_standard/filtered_func_data_clean -Tstd -bin reg_standard/filtered_func_data_clean_stdmask

    cd $origDir
else
    echo "No rfMRI directory for subject $1"
fi
```

## 4. functional connectivity

functional connectivity is the pearson correlation of pair regions' time series data. The final output is a symmetric matrix showing the correlation coefficient between two regions.

```bash
### register parcellation (labelled GM) to fMRI
# create inverse warp
${FSLDIR}/bin/convert_xfm -omat ${subjdir}/fMRI/rfMRI.ica/reg/highres2example_func.mat -inverse ${subjdir}/fMRI/rfMRI.ica/reg/example_func2highres.mat
# apply it to labelled GM
${FSLDIR}/bin/applywarp -i ${subjdir}/T1/labelled_GM -r ${subjdir}/fMRI/rfMRI.ica/example_func -o ${subjdir}/fMRI/rfMRI.ica/parcellation --premat=${subjdir}/fMRI/rfMRI.ica/reg/highres2example_func.mat --interp=nn

### segstate
mri_segstats --avgwf ${ts_rois} --i ${subjdir}/fMRI/rfMRI.ica/filtered_func_data_clean.nii.gz --seg ${subjdir}/fMRI/rfMRI.ica/parcellation.nii.gz  --sum ${stats_sum}

### FC coumpute
${BB_BIN_DIR}/bb_functional_pipeline/tvb_FC_compute -stat ${stats_sum} -ts $ts_rois -od ${subjdir}/fMRI/rfMRI.ica -LUT ${PARC_LUT}

```

