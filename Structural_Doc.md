## Steps in structural preprocessing using TVB-UKBB pipeline

T1 structural images are referred for all other fMRI modalities. A good quality of the structural images is vital for all subsequent processing.

Let's decomposite the TVB-UKBB pipeline in this documentation. We will start from the structural T1 weight imaging

### Workflow of the structural pipeline

1. Gradient distortion correction (GDC)
  GDC is to correct the gradient non-linearity in MRI machine. The principle of the MRI imaging is that, MRI machine will create a strong magnetic field (B0) so the hydrogen can orbit in the same/opposite directions (N-S, OR S-N, depends on the high or low energy state). We are living at a 3D world so there are three magnetic fields: X (Left–Right)	Y (Anterior–Posterior) and Z (Head–Foot). In order to localize the proton, each direction has  From a classical paper **J.Jovicich et al., 2015.**, ![image](https://user-images.githubusercontent.com/37648360/157306856-d8141fb7-02cb-49e5-8c60-98cbb609baa6.png)
