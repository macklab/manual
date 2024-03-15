# ASHS

[ASHS](https://sites.google.com/view/ashs-dox/home) is a fast method for automatically segmenting hippocampal subfields on a high-resolution T2-weighted volume. ASHS can be run locally on ix or through a cloud-based distributed segmentation service (DSS).

## ASHS on ix

Running ASHS on ix is simple. First, navigate to your project's BIDS directory. Then, activate the `mri-preproc` conda environment:

```
$ conda activate mri-preproc
```

Next, set an environment variable to the path of the ASHS binaries:

```
$ export ASHS_ROOT=/data/software/ashs/ashs
```

Finally, you can run ASHS on your participant. To do so, you will need the path to their T1 and T2 volumes and the path to the Princeton Young Adult atlas. Here, I'll use an example participant from the gardener project:

```
$ ${ASHS_ROOT}/bin/ashs_main.sh \
    -I sub-028 \
    -a /data/software/ashs/atlases/ashs_atlas_princeton \
    -g /data2/gardener/sub-028/anat/sub-028_T1w_crop.nii.gz \
    -f /data2/gardener/sub-028/anat/sub-028_T2w.nii.gz \
    -w /data2/gardener/derivatives/ashs/sub-028
```

In the above command, `-I` is the subject code, `-a` sets the path to the atlas (this is fixed, you won't change this!), `-g` and `-f` are the paths to the participant's T1 and T2 volumes, and `-w` is the output directory. 

ASHS takes 10-15 minutes to finish a segmentation. The output directory will contain all the files ASHS generates including measures of ICV and subfield volumes. The final segmentations and measures are in the `final` directory, look for the *_left_lfseg_corr_usegray.nii.gz and *_right_lfseg_corr_usegray.nii.gz volumes. The two hemispheres are segmented separately and will be in the same space as the T2 volume. They can be quickly combined with the following fslmaths command:

```
$ fslmaths sub-028_left_lfseg_corr_usegray.nii.gz -add sub-028_right_lfseg_corr_usegray.nii.gz sub-028_ashs.nii.gz
```

## ASHS via DSS

The ASHS documentation provides a [helpful tutorial](https://sites.google.com/view/ashs-dox/cloud-ashs/cloud-ashs-for-t2-mri) for how to use their cloud-based service. Note that we typically use the Princeton Young Adult atlas which is different from the atlas noted in the documentation. A few notes about DSS: It is free to use, works well, and can be run interactively with itk-SNAP which makes checking the quality of the segmentation straightforward. The downsides are that a limited number of segmentations can be performed at the same time and submitting jobs through itk-SNAP can be tedious with a larger number of participants.
