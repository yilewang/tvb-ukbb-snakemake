
```bash
cp T1_orig.nii.gz T1_orig_ud.nii.gz
```

Input: T1_orig.nii.gz  
Output: R1_orig_ud.nii.gz

```bash
#Calculate where does the brain start in the z dimension and then extract the roi
head_top=`${FSLDIR}/bin/robustfov -i T1_orig_ud | grep -v Final | head -n 1 | awk '{print $5}'`
${FSLDIR}/bin/fslmaths T1_orig_ud -roi 0 -1 0 -1 $head_top 170 0 1 T1_tmp
```

Input: T1_orig_ud.nii.gz  
Output: T1_tmp


```bash
#Run a (Recursive) brain extraction on the roi
${FSLDIR}/bin/bet T1_tmp T1_tmp_brain -R
```

Input: T1_tmp  
Output: T1_tmp_brain

```bash
#Reduces the FOV of T1_orig_ud by calculating a registration from T1_tmp_brain to ssref and applies it to T1_orig_ud. Keeps intermediate outputs for concatenation in next step (T1_tmp2_tmp_to_std.mat)
${FSLDIR}/bin/standard_space_roi T1_tmp_brain T1_tmp2 -maskNONE -ssref $FSLDIR/data/standard/MNI152_T1_1mm_brain -altinput T1_orig_ud -d
```
Input:T1_tmp_brain $FSLDIR/data/standard/MNI152_T1_1mm_brain (reference) T1_orig_ud (alternative reference)  
Output: T1_tmp2

`${FSLDIR}/bin/immv T1_tmp2 T1 #Rename T1_tmp2 to T1`

Input: T1_tmp2  
Output: T1

```bash
#Generate the actual affine from the orig_ud volume to the cut version we have now and combine it to have an affine matrix from orig_ud to MNI
${FSLDIR}/bin/flirt -in T1 -ref T1_orig_ud -omat T1_to_T1_orig_ud.mat -schedule $FSLDIR/etc/flirtsch/xyztrans.sch 
${FSLDIR}/bin/convert_xfm -omat T1_orig_ud_to_T1.mat -inverse T1_to_T1_orig_ud.mat
${FSLDIR}/bin/convert_xfm -omat T1_to_MNI_linear.mat -concat T1_tmp2_tmp_to_std.mat T1_to_T1_orig_ud.mat
```
Input: T1 T1_orig_ud (reference)  
output: T1_to_T1_orig_ud.mat (matrix)

Input: T1_to_T1_orig_ud.mat  
output: T1_orig_ud_to_T1.mat

Input: T1_tmp2_tmp_to_std.mat T1_to_T1_orig_ud.mat  
output: T1_to_MNI_linear.mat

```bash
#Non-linear registration of T1 to MNI using the previously calculated alignment
${FSLDIR}/bin/fnirt --in=T1 --ref=$FSLDIR/data/standard/MNI152_T1_1mm --aff=T1_to_MNI_linear.mat \
  --config=$BB_BIN_DIR/bb_data/bb_fnirt.cnf --refmask=$templ/MNI152_T1_1mm_brain_mask_dil_GD7 \
  --logout=../logs/bb_T1_to_MNI_fnirt.log --cout=T1_to_MNI_warp_coef --fout=T1_to_MNI_warp \
  --jout=T1_to_MNI_warp_jac --iout=T1_brain_to_MNI.nii.gz --interp=spline
```

Input: 
Output: T1_brain_to_MNI.nii.gz


