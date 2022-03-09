# Steps in structural preprocessing using TVB-UKBB pipeline

T1 structural images are referred for all other fMRI modalities. A good quality of the structural images is vital for all subsequent processing.

Let's decomposite the TVB-UKBB pipeline in this documentation. We will start from the structural T1 weight imaging.

The pipeline workflow of this process could be found in **F. Alfaro-Almagro et al., 2018** neuroimage paper.

![image](https://user-images.githubusercontent.com/37648360/157475349-307a3eb3-8574-4b8f-97af-6a5f9df0e7ae.png)


## Workflow of the structural pipeline

### 1. Gradient distortion correction (GDC)
GDC is to correct the gradient non-linearity in MRI machine. The principle of the MRI imaging is that, MRI machine will create a strong magnetic field (B0) so the hydrogen can orbit in the same/opposite directions (N-S, OR S-N, depends on the high or low energy state). We are living at a 3D world so there are three magnetic fields: X (Left–Right)	Y (Anterior–Posterior) and Z (Head–Foot). In order to localize the proton, each direction has a gradient from top to the end. However, the magnetic field gradient is not perfectly linear in the real world. Becasue of hardware limitation, there is always gradient non-linearity in the magnetic field. From a classical paper **J.Jovicich et al., 2015.**, they developed a new method to do the GDC and eliminate the geometric distortion in the MRI images (see the figure below). 

![image](https://user-images.githubusercontent.com/37648360/157306856-d8141fb7-02cb-49e5-8c60-98cbb609baa6.png).

```bash
#Calculate and apply the Gradient Distortion Unwarp ###NO GDC###
$BB_BIN_DIR/bb_pipeline_tools/bb_GDC --workingdir=./T1_GDC/ --in=T1_orig.nii.gz --out=T1_orig_ud.nii.gz --owarp=T1_orig_ud_warp.nii.gz
```

### 2. Cut down the FOV
The second step is to cut down the Field of view in the brain, to focus on brain tissues and improve quality of the registration. BET (brain extraction tool), FLIRT (FMRIB's linear image registration tool, and MNI152 "non-linear 6th generation" standard-space T1 template will be used in this step. 


```bash
# to calculate the length of the whole brain in z dimension
#Calculate where does the brain start in the z dimension and then extract the roi
head_top=`${FSLDIR}/bin/robustfov -i T1_orig_ud | grep -v Final | head -n 1 | awk '{print $5}'`
${FSLDIR}/bin/fslmaths T1_orig_ud -roi 0 -1 0 -1 $head_top 170 0 1 T1_tmp

#Run a (Recursive) brain extraction on the roi
${FSLDIR}/bin/bet T1_tmp T1_tmp_brain -R

#Reduces the FOV of T1_orig_ud by calculating a registration from T1_tmp_brain to ssref and applies it to T1_orig_ud. Keeps intermediate outputs for concatenation in next step (T1_tmp2_tmp_to_std.mat)
${FSLDIR}/bin/standard_space_roi T1_tmp_brain T1_tmp2 -maskNONE -ssref $FSLDIR/data/standard/MNI152_T1_1mm_brain -altinput T1_orig_ud -d

# Rename the T1_tmp2 to T1
${FSLDIR}/bin/immv T1_tmp2 T1

#Generate the actual affine from the orig_ud volume to the cut version we have now and combine it to have an affine matrix from orig_ud to MNI
${FSLDIR}/bin/flirt -in T1 -ref T1_orig_ud -omat T1_to_T1_orig_ud.mat -schedule $FSLDIR/etc/flirtsch/xyztrans.sch 
${FSLDIR}/bin/convert_xfm -omat T1_orig_ud_to_T1.mat -inverse T1_to_T1_orig_ud.mat
${FSLDIR}/bin/convert_xfm -omat T1_to_MNI_linear.mat -concat T1_tmp2_tmp_to_std.mat T1_to_T1_orig_ud.mat
```


### 3. Non-linear registration to MNI152 space using FNIRT (FMRIB's non-linear image registration tool). 
All of the transformations estimated in GDC, linear and non-linear transformations to MNI152 are combined into one single non-linear transformation and allows the original T1 to be transformed into MNI152 space

```bash
#Non-linear registration of T1 to MNI using the previously calculated alignment
${FSLDIR}/bin/fnirt --in=T1 --ref=$FSLDIR/data/standard/MNI152_T1_1mm --aff=T1_to_MNI_linear.mat \
  --config=$BB_BIN_DIR/bb_data/bb_fnirt.cnf --refmask=$templ/MNI152_T1_1mm_brain_mask_dil_GD7 \
  --logout=../logs/bb_T1_to_MNI_fnirt.log --cout=T1_to_MNI_warp_coef --fout=T1_to_MNI_warp \
  --jout=T1_to_MNI_warp_jac --iout=T1_brain_to_MNI.nii.gz --interp=spline
  
```



### 4. Inverse of the MNI152
a standard-space brain mask is transformed into the native T1 space and applied to the T1 image to generate a brain-extracted T1. 

```bash
#Create brain mask and mask T1
${FSLDIR}/bin/invwarp --ref=T1 -w T1_to_MNI_warp_coef -o T1_to_MNI_warp_coef_inv
${FSLDIR}/bin/applywarp --rel --interp=nn --in=$templ/MNI152_T1_1mm_brain_mask --ref=T1 -w T1_to_MNI_warp_coef_inv -o T1_brain_mask
${FSLDIR}/bin/fslmaths T1 -mul T1_brain_mask T1_brain
${FSLDIR}/bin/fslmaths T1_brain_to_MNI -mul $templ/MNI152_T1_1mm_brain_mask T1_brain_to_MNI

#register parcellation, cerebellar & brainstem masks to T1
${FSLDIR}/bin/applywarp --rel --interp=nn --in=$PARC_IMG --ref=T1 -w T1_to_MNI_warp_coef_inv -o parcel_to_T1
${FSLDIR}/bin/applywarp --rel --interp=nn --in=${templ}/cerebellum --ref=T1 -w T1_to_MNI_warp_coef_inv -o cerebellum_to_T1
${FSLDIR}/bin/applywarp --rel --interp=nn --in=${templ}/brainstem --ref=T1 -w T1_to_MNI_warp_coef_inv -o brainstem_to_T1
```

### 5. Defacing process
This step is for anonymous purpose

```bash
#Defacing T1
${FSLDIR}/bin/convert_xfm -omat grot.mat -concat T1_to_MNI_linear.mat T1_orig_ud_to_T1.mat
${FSLDIR}/bin/convert_xfm -omat grot.mat -concat $templ/MNI_to_MNI_BigFoV_facemask.mat grot.mat
${FSLDIR}/bin/convert_xfm -omat grot.mat -inverse grot.mat
${FSLDIR}/bin/flirt -in $templ/MNI152_T1_1mm_BigFoV_facemask -ref T1_orig -out grot -applyxfm -init grot.mat
#${FSLDIR}/bin/fslmaths grot -mul -1 -add 1 -mul T1_orig T1_orig_defaced
${FSLDIR}/bin/fslmaths grot -binv -mul T1_orig T1_orig_defaced

cp T1.nii.gz T1_not_defaced_tmp.nii.gz  
${FSLDIR}/bin/convert_xfm -omat grot.mat -concat $templ/MNI_to_MNI_BigFoV_facemask.mat T1_to_MNI_linear.mat
${FSLDIR}/bin/convert_xfm -omat grot.mat -inverse grot.mat
${FSLDIR}/bin/flirt -in $templ/MNI152_T1_1mm_BigFoV_facemask -ref T1 -out grot -applyxfm -init grot.mat
${FSLDIR}/bin/fslmaths grot -binv -mul T1 T1


# We don't touch this part because tvb-ukbb has another system dedicated for QC
#Generation of QC value: Number of voxels in which the defacing mask goes into the brain mask
#${FSLDIR}/bin/fslmaths T1_brain_mask -thr 0.5 -bin grot_brain_mask 
#${FSLDIR}/bin/fslmaths grot -thr 0.5 -bin -add grot_brain_mask -thr 2 grot_QC
#${FSLDIR}/bin/fslstats grot_QC.nii.gz -V | awk '{print $ 1}' > T1_QC_face_mask_inside_brain_mask.txt

```


### 6. Tissue-type segmentation FAST (FMRIB's Automated Segmentation Tool). 
This step is to create a T1 image with bias-field-correction, to discrete CSF, grey matter and white matter


```bash
#Run fast
mkdir T1_fast
${FSLDIR}/bin/fast -b -o T1_fast/T1_brain T1_brain 

# pve masks: Partial Volume Estimates, create mask for CSF, Grey Matter and White Matter
#Binarize PVE masks
if [ -f T1_fast/T1_brain_pveseg.nii.gz ] ; then
    $FSLDIR/bin/fslmaths T1_fast/T1_brain_pve_0.nii.gz -thr 0.5 -bin T1_fast/T1_brain_CSF_mask.nii.gz
    $FSLDIR/bin/fslmaths T1_fast/T1_brain_pve_1.nii.gz -thr 0.5 -bin T1_fast/T1_brain_GM_mask.nii.gz
    $FSLDIR/bin/fslmaths T1_fast/T1_brain_pve_2.nii.gz -thr 0.5 -bin T1_fast/T1_brain_WM_mask.nii.gz
fi

#remove cerebellum and brainstem from GM and WM mask
${FSLDIR}/bin/fslmaths T1_fast/T1_brain_GM_mask -sub transforms/cerebellum_to_T1 -sub transforms/brainstem_to_T1 -bin T1_fast/T1_brain_GM_mask_noCerebBS
${FSLDIR}/bin/fslmaths T1_fast/T1_brain_WM_mask -sub transforms/cerebellum_to_T1 -sub transforms/brainstem_to_T1 -bin T1_fast/T1_brain_WM_mask_noCerebBS


#Apply bias field correction to T1
if [ -f T1_fast/T1_brain_bias.nii.gz ] ; then
    ${FSLDIR}/bin/fslmaths T1.nii.gz -div T1_fast/T1_brain_bias.nii.gz T1_unbiased.nii.gz
    ${FSLDIR}/bin/fslmaths T1_brain.nii.gz -div T1_fast/T1_brain_bias.nii.gz T1_unbiased_brain.nii.gz
else
    echo "WARNING: There was no bias field estimation. Bias field correction cannot be applied to T1."
fi

```





### 7. Subcortical structures modeled by FIRST (FMRIB's Integrated Registration and Segmentation tool)
The shape and volume output for 15 subcortical regions are generated and stored. 

```bash
#Run First
mkdir T1_first

#Creates a link inside T1_first to ./T1_unbiased_brain.nii.gz (In the present working directory)
ln -s ../T1_unbiased_brain.nii.gz T1_first/T1_unbiased_brain.nii.gz
${FSLDIR}/bin/run_first_all -i T1_first/T1_unbiased_brain -b -o T1_first/T1_first

#add FIRST subcortical segmentation to GM segmentation without brainstem
${FSLDIR}/bin/fslmaths T1_first/T1_first_all_fast_firstseg -uthr 15 T1_first/subcort_upperthresh_seg
${FSLDIR}/bin/fslmaths T1_first/T1_first_all_fast_firstseg -thr 17 T1_first/subcort_lowerthresh_seg
${FSLDIR}/bin/fslmaths T1_first/subcort_upperthresh_seg -add T1_first/subcort_lowerthresh_seg -bin T1_first/subcort_GM
${FSLDIR}/bin/fslmaths T1_fast/T1_brain_GM_mask_noCerebBS -add T1_first/subcort_GM -bin cort_subcort_GM
```

### 8. SIENAX analysis & VMB
normalise brain tissue volumes for head size. volumnes generation

```bash
#Run Siena
$BB_BIN_DIR/bb_structural_pipeline/bb_sienax `pwd`/..


#Run VBM
$BB_BIN_DIR/bb_structural_pipeline/bb_vbm `pwd`/..

```



### 9. BIANCA (Brain Intensity AbNormality Classification Algorithm) is a tool to automatically differentiate white matter hyperintensities. 



```bash
#Run BIANCA
$BB_BIN_DIR/bb_structural_pipeline/bb_BIANCA $1

#remove BIANCA estimated lesion voxels from GM masks & add to WM mask
cp $1/T1/T1_fast/T1_brain_GM_mask.nii.gz $1/T1/T1_fast/T1_brain_GM_mask_preBIANCA.nii.gz
${FSLDIR}/bin/fslmaths $1/T1/T1_fast/T1_brain_GM_mask_preBIANCA -sub $1/T2_FLAIR/lesions/final_mask -bin $1/T1/T1_fast/T1_brain_GM_mask
cp $1/T1/T1_fast/T1_brain_GM_mask_noCerebBS.nii.gz $1/T1/T1_fast/T1_brain_GM_mask_noCerebBS_preBIANCA.nii.gz
${FSLDIR}/bin/fslmaths $1/T1/T1_fast/T1_brain_GM_mask_noCerebBS_preBIANCA -sub $1/T2_FLAIR/lesions/final_mask -bin $1/T1/T1_fast/T1_brain_GM_mask_noCerebBS
cp $1/T1/cort_subcort_GM.nii.gz $1/T1/cort_subcort_GM_preBIANCA.nii.gz
${FSLDIR}/bin/fslmaths $1/T1/cort_subcort_GM_preBIANCA -sub $1/T2_FLAIR/lesions/final_mask -bin $1/T1/cort_subcort_GM
cp $1/T1/T1_fast/T1_brain_WM_mask.nii.gz $1/T1/T1_fast/T1_brain_WM_mask_preBIANCA.nii.gz
${FSLDIR}/bin/fslmaths $1/T1/T1_fast/T1_brain_WM_mask_preBIANCA -add $1/T2_FLAIR/lesions/final_mask -bin $1/T1/T1_fast/T1_brain_WM_mask
cp $1/T1/T1_fast/T1_brain_WM_mask_noCerebBS.nii.gz $1/T1/T1_fast/T1_brain_WM_mask_noCerebBS_preBIANCA.nii.gz
${FSLDIR}/bin/fslmaths $1/T1/T1_fast/T1_brain_WM_mask_noCerebBS_preBIANCA -add $1/T2_FLAIR/lesions/final_mask -bin $1/T1/T1_fast/T1_brain_WM_mask_noCerebBS

#create GM-WM interface for dMRI tractography seeds
${FSLDIR}/bin/fslmaths $1/T1/T1_fast/T1_brain_GM_mask_noCerebBS -kernel sphere 1 -dilM $1/T1/T1_fast/T1_brain_GM_mask_noCerebBS_dil
${FSLDIR}/bin/fslmaths $1/T1/T1_fast/T1_brain_GM_mask_noCerebBS_dil -add $1/T1/T1_fast/T1_brain_WM_mask_noCerebBS -thr 2 -bin $1/T1/interface

#check interface, re-do with extra dilation if empty
maskMAX=$(fslstats $1/T1/interface -R | awk '{print int($2)}')
if [ $maskMAX -lt 1 ]
then
     echo "interface mask empty, dilating GM mask by 2 and re-doing interface"
     ${FSLDIR}/bin/fslmaths $1/T1/T1_fast/T1_brain_GM_mask_noCerebBS -kernel sphere 2 -dilM $1/T1/T1_fast/T1_brain_GM_mask_noCerebBS_dil2
     ${FSLDIR}/bin/fslmaths $1/T1/T1_fast/T1_brain_GM_mask_noCerebBS_dil2 -add $1/T1/T1_fast/T1_brain_WM_mask_noCerebBS -thr 2 -bin $1/T1/interface
fi
```


### 10. label the grey matter with ROIs

```bash
#label GM with ROIs
${AFNIDIR}/3dROIMaker -inset $1/T1/cort_subcort_GM.nii.gz -thresh 0.1 -inflate 1 -prefix $1/T1/labelled -refset $1/T1/transforms/parcel_to_T1.nii.gz -nifti -neigh_upto_vert -dump_no_labtab

#for some reason both labelled_GM and labelled_GMI are inflated; fix it here
${FSLDIR}/bin/fslmaths $1/T1/labelled_GM -mas $1/T1/cort_subcort_GM.nii.gz $1/T1/labelled_GM
```



