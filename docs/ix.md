# Compute server: ix

## CPU cores
ix has a 20-core processor with hyper threading. This means there are a total of 40 cores available to the system for running processes (hyper threading looks better than it is, a set of hyper threading cores runs about 120% faster than a set of cores without hyper threading). So, that might seem like a lot of cores until eight people are trying to run their analyses all at the same time. In general, it is good to limit the number of cores available to any process. This is accomplished in a lot of different ways. Here's a list (that we should keep adding to):
* Environment variables: Setting OMP_NUM_THREADS in the command line ($ export OMP_NUM_THREADS=10) will some times work to limit the number of threads that can be opened by a single process. This requires that the process is multi-processor aware.
* pymvpa: Multiple cores are critical for searchlight analyses. This requires two things, first setting OMP_NUM_THREADS equal to 1 in the terminal, and second running pymvpa searchlight code that sets nproc to some value in the range 4-10.
* MRtrix has its own flag for setting the number of threads in all of this functions. Add -nthreads N, where N is 4-10, in any MRtrix command.

## Accessing data drives over SMB

Users can remote connect to ix data drives via SMB such that they are mounted locally through Finder (or god forbid Windows Explorer). To do so, you first need to create a SMB password on ix. Log in via ssh, then run the following command:
```
$ sudo smbpasswd -a <username>
```
Type in your smb password and remember it! Then, you need to be added to the sambashare group. Run the `groups` command from the command line to see the list of groups you belong to. If sambashare is not in the list, ask Mike or a grad student to add you. Finally, on your local computer, create a new remote connection to ix. Here's how to do it in Finder on MacOS: From the Go menu, select Connect to Server... (or hit commmand+k), type in "smb://ix.mack.psych.utoronto.ca" in the address bar on top, and click Connect. For the login information, select "Registered User" and use your ix username and the SMB password you just set. Check the box to remember the credentials so you don't have to put this in every time you connect. Pick which data drive you want to mount and you'll have direct access to your files on ix. 

## Managing projects on ix

Projects are stored on ix in the /data and /data2 drives. Each project has a main directory (e.g., `/data2/<project name>`). Each project is managed by a specific user (or maybe a few users). To be a project manager, your user account will need sudo access. If you don't already have sudo access (or know what this is), ping Mike! 

### File access policies on ix
To encourage coding sharing and collaboration but also protect our project files on ix, we have implemented the following policies for file/directory access:

1. All projects are world readable. This means any user on ix can view and copy files/directories from all projects.
2. Project files/directories can be modified (i.e., create, update, delete operations) by only those users who are members of project groups. Each project will have its own unix group and users of that project will have to be added to the project group to modify the project files.
3. Backups for project files are opt-in. Project managers will have to manually add project directories to the main ix backup list.

### Setting up a new project

New projects on ix require some setup. Follow the instructions below to manage a new project. Note: project management requires a user account with sudo access. If you don't already have sudo access, ping Mike! Also, all of the commands listed below assume you are logged into ix and using a terminal. The `$` represents a command line prompt

#### Create a project directory
Each project will have its own base directory. All files/directories related to this project should be stored within this base directory. This would include subdirectories for pilots as well as the a project's core studies and analyses. Create a new project directory with the following command, where `<projectname>` is a placeholder for your project's name:

```
$ mkdir /data2/<projectname>
```

#### Create a project group
Your new project needs a user group! Note, you will need to use the `sudo` command to create a group. `sudo` gives you temporary elevated privileges while running a command and requires that you enter your password before the command is completed. To keep things easy, let's use the project's name for the project group. Here's how to create a group:

```
$ sudo groupadd <projectname>
```

You can check that the group has been created by running the following command (here we are using funclearn as an example project):

```
$ getent group | grep funclearn
funclearn:x:1034:
```
You should see a line of text that shows the group name and its internal unique group ID (i.e., 1034 in the example). If you see this, your group has been created.

### Set project directory's group and SetGID bit
The project directory's group needs to be set to the project group:

```
$ sudo chgrp -R <projectname> /data2/<projectname>
```

A few notes for the command above: The order of the two arguments for `chgrp` are first the group name and second the directory path. Also, the `-R` flag says to additionally change the group for all files/directories contained in the directory. If you just created your project directory and it is empty, the `-R` flag isn't necessary.

To check that your project directory's group has been changed, run this `ls` command (again, using funclearn as an example):

```
$ ls -ld /data2/funclearn
drwxrwxr-x+ 47 mmack funclearn 4096 May 10 11:32 /data2/funclearn
```

Quick description of the output: The very first `d` says that this is a directory. The next string of characters define file permissions for the directory (`r`=read, `w`=write, `x`=execute) for the file owner (first three characters), file group (second three characters), and everyone else (last three characters). If you see the `r`, `w`, `x`, it means that permission for that user or group is set, otherwise (i.e., `-`) that access is not set. Skip over the number and then the next two columns list the directory owner (`mmack`) and group (`funclearn`). If you see your project name in the group column, your project directory's group is all set!

Last thing to worry about here is to set the project directory's SetGID bit. "The what?", you ask. The SetGID bit on a directory, if set, means that any file or directory created within that directory will have the same group. This is key, we want all of the files/directories within your project directory to have the same project group. To do set the SetGID bit, run this command:

```
$ sudo chmod g+s /data2/<projectname>
```

Let's check what happened with that same `ls` command (using funclearn as an example):

```
$ ls -ld /data2/funclearn
drwxrwsr-x+ 47 mmack funclearn 4096 May 10 11:32 /data2/funclearn
```

There's one change that should have happened: Look for the `s` in the group permission characters. The `s` has replaced the `x`. This means that SetGID bit has been set and you are in good shape.

!!! note "Existing projects"

    If you are managing a project with data files already on ix (e.g., you are updating an existing project), you will need to change the group for all of the files/directories in the project directory. To do this, run this command (using funclearn as an example):

    ``` 
    $ sudo chgrp -R funclearn /data2/funclearn
    ```

### Add users to the project group
Adding users to the project group is necessary for them to be able to modify the project files/directories. To add a user to the project group, run this command (using funclearn as an example):

```
$ sudo usermod -a -G funclearn <username>
```

The `-a` flag says to append the group to the user's existing groups. The `-G` flag says to add the user to the group. The last argument is the user's username. You can check that the user has been added to the group by running this command (using funclearn as an example):

```
$ groups <username>
```

You should see the project group listed in the output. If you don't, try logging out and logging back in. You could also look at the output of the `getent` command from above to see if the user is listed in the project group:

```
$ getent group | grep funclearn
funclearn:x:1034:mmack,<username>
```

Don't forget to add yourself to the project group!

### Adding project files to ix backup

ix has a backup system that runs every night. In order to keep things running smoothly, we have to limit the number of files that are backed up. If configured to do so, this system will backup all files/directories from a project except for those that ar easily created with standard pipelines. As policy, we will **not** backup the following files/directories:

* Raw nifti files (e.g., sub-XXX in the root BIDS directory). These files are easily recreated from the raw DICOM files which are backed up to Arrakis or Trellis. 
* fmriprep output. These files are easily recreated from the raw nifti files and the specific version of fmriprep you are using for your project.
* Freesurfer output. These files are easily recreated from the raw nifti files and the specific version of freesurfer you are using (likely bundled with fmriprep).
* Any temporary files/directories (e.g., tmp_dcm2bids)

By default, we will include everyting else in the project directory in the backup. 

To add your project to the backup list:

1. Ping Mike with your project name and root directory (e.g., /data2/funclearn)
2. Create a file named `backup_exclude.txt` in the root directory of your project. This file should contain a list of files/directories that you want to exclude from the backup. Each file/directory should be on its own line. Here's an example from funclearn which includes all the noted exclusions from above:
```
/data2/funclearn/sub-*/
/data2/funclearn/sourcedata/
/data2/funclearn/tmp_dcm2bids/
/data2/funclearn/derivatives/fmriprep/
/data2/funclearn/derivatives/freesurfer/
```
The file/directory paths must be absolute paths. Also, the `*` is a wildcard character that matches any string. So, `sub-*` will match any file/directory that starts with `sub-` (i.e., the raw nifti files for each participant).

Important note: If your project directory is a BIDS root, you will need to add `backup_exclude.txt` to your `.bidsignore` file to pass BIDS validation checks.

## Docker on ix
To ensure that our file permissions setup works with Docker, we need to specify in docker calls which user and group ID to use while that call is completed. Setting these options means that any files and directories that are created by the docker call will have the permissions of the user and group ID specified. If this isn't set, all files/directories are owned by root. Here's how to set the user and group IDs.

First, you need your user ID and the project group ID for the data you are working on. These IDs are numbers associated with user and group accounts. An easy way to retrieve this is to run the `id` command, which lists both your user id (i.e., uid) and all the IDs for the groups you belong to:
```
$ id
uid=1000(mmack) gid=1000(mmack) groups=1000(mmack),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd),120(sambashare),1001(macklab),1011(bearholiday),1012(funclearn),1014(bighipp),1015(birthday),1016(clusterbake),1019(difflearn),1020(dtilearn),1021(eyelearn),1024(gardener),1027(videoschema),1030(em-hddm),1031(hippcircuit),1032(docker),1033(scenewheel),1034(aa_diffodd)
```
Second, you should add a `--user` argument to your docker calls. For example, if I wanted to run fmriprep via docker on the funclearn data, my docker call would look like this:
```
$ docker run -it --rm --user 1000:1012 \
    -v /data/software/fs_license.txt:/opt/freesurfer/license.txt \
    -v /data2/templateflow:/templateflow \
    -v /data2/funclearn:/data:ro \
    -v /data2/funclearn/derivatives/fmriprep:/out \
    nipreps/fmriprep:23.1.4 \
    /data /out participant \
    --ignore slicetiming \
    --participant_label 100 \
    --nthreads 10 \
    --notrack \
    --skip-bids-validation \
    --output-spaces MNI152NLin2009cAsym func
```
Note the `--user 1000:1012` argument in the first line. This sets the user to be me (uid=1000) and group to be funclearn (group ID = 1012) with these numbers pulled from the `id` command above. By settingi the `--user` argument, all files and directories created by the fmriprep call will have me as the owner and funclearn as the group. 

NOTE: If you are using a newer version of fmriprep (ver 22.0 and newer), you must explictly define the name of the output directory inside the derivatives directory (e.g., `-v /data2/funclearn/derivatives/fmriprep:/out`). Prior to this, fmriprep defaulted to creating a directory named fmriprep. To make this all work well with our permissions setup, please create your fmriprep derivatives folder first before running fmriprep on your participants. If you don't, the output folder will be created by docker with root as owner and group. If this happens, all is not lost. You can change the group to your project group (e.g., `$ sudo chgrp -R funclearn /data2/funclearn/derivatives/fmriprep`), then subsequent fmriprep calls will work without write permission errors.

