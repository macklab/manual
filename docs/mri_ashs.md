# ASHS

[ASHS](https://sites.google.com/view/ashs-dox/home) is a fast method for automatically segmenting hippocampal subfields on a high-resolution T2-weighted volume. ASHS can be run locally on ix or through a cloud-based distributed segmentation service (DSS).

## ASHS on ix

Running ASHS on ix is simple. First, activate the `mri-preproc` conda environment:

'''sh
> conda activate mri-preproc
'''

## ASHS via DSS

The ASHS documentation provides a [helpful tutorial](https://sites.google.com/view/ashs-dox/cloud-ashs/cloud-ashs-for-t2-mri) for how to use their cloud-based service. Note that we typically use the Princeton Young Adult atlas which is different from the atlas noted in the documentation. A few notes about DSS: It is free to use, works well, and can be run interactively with itk-SNAP which makes checking the quality of the segmentation straightforward. The downsides are that a limited number of segmentations can be performed at the same time and submitting jobs through itk-SNAP can be tedious with a larger number of participants.
