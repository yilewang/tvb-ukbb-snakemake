# Steps in structural preprocessing using TVB-UKBB pipeline

T1 structural images are referred for all other fMRI modalities. A good quality of the structural images is vital for all subsequent processing.

Let's decomposite the TVB-UKBB pipeline in this documentation. We will start from the structural T1 weight imaging

### Workflow of the structural pipeline

1. Gradient distortion correction (GDC)
GDC is to correct the gradient non-linearity in MRI machine. The principle of the MRI imaging is that, MRI machine will create a strong magnetic field (B0) so the hydrogen can orbit in the same/opposite directions (N-S, OR S-N, depends on the high or low energy state). We are living at a 3D world so there are three magnetic fields: X (Left–Right)	Y (Anterior–Posterior) and Z (Head–Foot). In order to localize the proton, each direction has a gradient from top to the end. However, the magnetic field gradient is not perfectly linear in the real world. Becasue of hardware limitation, there is always gradient non-linearity in the magnetic field. From a classical paper **J.Jovicich et al., 2015.**, they developed a new method to do the GDC and eliminate the geometric distortion in the MRI images (see the figure below). 

![image](https://user-images.githubusercontent.com/37648360/157306856-d8141fb7-02cb-49e5-8c60-98cbb609baa6.png).
  
2. Cut down the FOV
The second step is to cut down the Field of view in the brain, to focus on brain tissues and improve quality of the registration. BET (brain extraction tool), FLIRT (FMRIB's linear image registration tool, and MNI152 "non-linear 6th generation" standard-space T1 template will be used in this step. 

3. Non-linear registration to MNI152 space using FNIRT (FMRIB's non-linear image registration tool). 
All of the transformations estimated in GDC, linear and non-linear transformations to MNI152 are combined into one single non-linear transformation and allows the original T1 to be transformed into MNI152 space

4. Inverse of the MNI152
a standard-space brain mask is transformed into the native T1 space and applied to the T1 image to generate a brain-extracted T1. 

5. Defacing process

6. Tissue-type segmentation FAST (FMRIB's Automated Segmentation Tool). 
This step is to create a T1 image with bias-field-correction, to discrete CSF, grey matter and white matter

7. SIENAX analysis
normalise brain tissue volumes for head size. volumnes generation

8. Subcortical structures modeled by FIRST (FMRIB's Integrated Registration and Segmentation tool)
The shape and volume output for 15 subcortical regions are generated and stored. 

9. list of the volume measures from SIENAX: total brain (grey + white matter), total white matter volume, total grey matter volum, ventricular, CSF, perpheral cortical grey matter. 
