# Using Beluga HPC system for fmriprep

[Beluga](https://docs.alliancecan.ca/wiki/B%C3%A9luga/en) is one of the HPC systems under the Digital Research Alliance umbrella. We have access to it without the need for a resource allocation award and it provides 1TB of project space for each PI (i.e., the Mack Lab has 1TB). That's not a ton of space, but it is enough for temporary storage while running fmriprep jobs.

## Logging into Beluga
Beluga uses the same login credentials as all of the Alliance systems. You can access the login node with SSH via the command line or Visual Studio Code: 

``` bash
$ ssh <username>@beluga.computecanada.ca
```

Setting up [SSH keys through the CCDB](https://docs.alliancecan.ca/wiki/SSH_Keys) is highly recommended so you don't have to worry about entering passwords.

## Initial setup
We will use apptainer images of fmriprep docker images which means initial setup is minimal. The main requirements are:

1. **fmriprep apptainer images:** Docker images of fmriprep versions have to be converted into Apptainer images. If you want to use a new version of fmriprep, follow the instructions from the [Alliance](https://docs.alliancecan.ca/wiki/Singularity) and [fmriprep](https://fmriprep.org/en/1.5.8/singularity.html#preparing-a-singularity-image). Available images are stored on Beluga in `/project/def-mmack/software/containers`.
2. **TemplateFlow:** TemplateFlow is a helper tool that allows programmatically access to standard neuroimaging templates on the fly (i.e., new templates are downloaded from the internet as needed). fmriprep depends on templateflow to work. This presents a problem for HPC systems as most setups, including Beluga, do not allow compute nodes to access the internet. As such, TemplateFlow images must be downloaded separately and made available to fmriprep. This repository is stored on Beluga in `/project/def-mmack/software/TemplateFlow`.

## Setup your BIDS dataset
You will need to copy your BIDS dataset to Beluga. Copy to your lab's project space (e.g., /project/def-mmack/<username> or /project/def/mschlich/<username>, or see below for an alternative location!). You can use scp commands or sftp apps to do so. You will not want to transfer all of your participants' data at once so as to not fill up the 1TB of project space. But, you need a valid BIDS dataset, so should include code, derivatives, and sub-XXX directories as well as the CHANGES, README, dataset_description.json, participants.json, participants.tsv, and .bidsignore files. 

An alternative is to use your scratch space for storing your BIDS dataset. Scratch is a huge but temporary disk space. Any files that are there for longer than 60 days without being modified will be deleted. If you use scratch, you have to be sure to copy results off of Beluga before they are deleted. On Beluga, scratch space is found under the /scratch path (e.g., `/scratch/mmack/`).

## Add and modify run_fmriprep.sh
The `run_fmriprep.sh` script included here is an example of how to make the apptainer call to run fmriprep on a single participant. This script should be copied into your BIDS dataset's code directory. 

The script needs to be updated to reflect your dataset location and any specific parameters you need for fmriprep. At a minimum, you should change the path to your dataset location and confirm which version of fmriprep you want to use which are defined in the script's first two variables:

``` bash title="run_fmriprep.sh"
#/bin/bash

# location of your BIDS dataset (CHANGE THIS!)
BIDSDIR=/project/def-mmack/projects/funclearn

# fmriprep version 
fpver=23.1.4

# location of fmriprep and templateflow
SWDIR=/project/def-mmack/software
TEMPLATEFLOW_HOME=${SWDIR}/templateflow
# paths for singularity image
export APPTAINERENV_FS_LICENSE=/freesurfer_license.txt
export APPTAINERENV_TEMPLATEFLOW_HOME=/templateflow
# create temporary directory
mkdir -p ${SCRATCH}/tmp

# run fmriprep
apptainer run --cleanenv \
  -B ${SWDIR}/freesurfer_license.txt:/freesurfer_license.txt \
  -B ${TEMPLATEFLOW_HOME:-$HOME/.cache/templateflow}:/templateflow \
  -B ${BIDSDIR}:/data \
  -B ${BIDSDIR}/derivatives:/out \
  -B ${SCRATCH}/tmp:/work \
  ${SWDIR}/containers/fmriprep-${fpver}.sif \
  /data /out \
  participant \
  --participant-label $1 \
  --work-dir /work \
  --ignore slicetiming \
  --fs-no-reconall \
  --nthreads 10 \
  --notrack \
  --output-spaces MNI152NLin2009cAsym T1w
```

## Submit fmriprep jobs
To submit a job to Beluga's scheduler, we need a second script: `submit_fmriprep.sh`: 

``` bash title='submit_fmriprep.sh'
#!/bin/bash
#SBATCH --account=def-mmack (MAYBE CHANGE THIS!)
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=10
#SBATCH --mem-per-cpu=4G
#SBATCH --time=2:00:00 (MAYBE CHANGE THIS!)
#SBATCH --job-name=fmriprep
#SBATCH --output=/project/def-mmack/mmack/funclearn/code/logs/fmriprep_%j.txt (CHANGE THIS!)
#SBATCH --mail-user=<your email address> (CHANGE THIS!)
#SBATCH --mail-type=ALL

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

module load apptainer

# location of your BIDS dataset (CHANGE THIS!)
PROJDIR=/project/def-mmack/projects/funclearn

${PROJDIR}/code/run_fmriprep.sh $1
```

This script needs a bit more modification (look for the CHANGE THIS! comments in the code above), specifically change:

* `--account` to your PI's project name (e.g., def-mmack or def-mschlich)
* `--time` to however much time your fmriprep run needs (see note below)
* `--output` to a log file path in your BIDS directory
* `--mail-user` to your email address
* `PROJECTDIR` to your BIDS dataset path  

!!! note "What time duration to use?"
    To decide what time duration to use, you want to find a safe minimal duration that isn't too long (otherwise your job will be stuck on the scheduler for longer) but provides enough time to finish the fmriprep job. There will be some variability across participants, so your time duration should have some buffer. Also, depending on what you plan to do with fmriprep can have a huge difference in timing (e.g., including freesurfer in the pipeline can add ~6 hours). Benchmark two participants by setting the time to a higher duration (e.g., 8-10 hours) and see how long the jobs takes to complete. Then take the maximum time and add 0.5-1 hour for your time parameter. As a point of reference, a default fmriprep job without freesurfer for one participant with 1 T1w, 1 fieldmap, 8 BOLD timeseries (~150 volumes each) takes approximately 1 hr 15 mins with 10 cores and 4GB RAM per core (i.e., as defined in the submit_fmriprep.sh script).  

With both of scripts updated and copied to your BIDS dataset's code directory, you should be all set to submit a job. To do so, while logged into Beluga, go to your BIDS directory and run the submit_fmriprep.sh script with a specific participant number, e.g.:

``` bash
$ sbatch code/submit_fmriprep.sh 100
```


If all goes well, a job will be added to the queue.

## Checking job status
There are a few tools for checking the status of running jobs. To get general stats about your running jobs, run the `sq` command while logged into Beluga. 

You can also watch a running job by attaching to the node with `srun` and running htop. First, run `sq` to get the jobid (i.e., first column of the `sq` output). Then, run:

``` bash
$ srun --jobid 123456 --pty htop -u $USER
```

## Job complete tasks
Once your job finishes, copy the fmriprep output in the derivaties folder back to your computer, Arrakis/Trellis, or ix. Finally, remove those files from Beluga, especially if you're using lab project space!
