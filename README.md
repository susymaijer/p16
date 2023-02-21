# p16

Repository for LUMC P16 continuous learning

# Installation

## Set-up virtual environment
First, we need to set-up a virtual environment to contain all the packages. I used venv since I had a bad fight with conda when my home directory went over the size limit (10GB). There are two options.

**Option 1: Set up from scratch (Recommended)**

I used the following commands:

> module purge

> module add system/python/3.10.2   -- replace with latest python version

> python3 -m venv [YOUR_VENV_NAME]

On default, environments get put in your home directory. We don't want this, since environments can take up a lot of space and there is a hard max of 10GB in your home directory. So specify another location where there is not space limit:

> mkdir /exports/lkeb-hpc/[YOUR_USER_NAME]/venv_environments

> pip config set pancreas.target /exports/lkeb-hpc/[YOUR_USER_NAME]/venv_environments/[YOUR_VENV_NAME]

Then, install the following packages:
- SimpleITK
- pydicom

**Option 2: Clone my venv environment (Not recommended)**

I'm not even sure if this would work, since the nnU-Net installation is quite specific. However, I added my venv enviromnent as a yaml file in TODO/TODO.

## Install nnU-Net

Follow the installation steps from https://github.com/MIC-DKFZ/nnUNet . I took the following steps, but **please make sure that it still matches the description from the nnU-net repository!**

First, activate your virtual environment: 

> source /exports/lkeb-hpc/[YOUR_USER_NAME]/venv_environments/[YOUR_VENV_NAME]/bin/activate

Then install torch using pip:

> pip3 install --upgrade --force-reinstal torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/cu116 -- use the latest CUDA version!!

Verify versions: 

> python -c 'import torch;print(torch.backends.cudnn.version())'

> python -c 'import torch;print(torch.__version__)'

Now, ready to install the nnU-net. Since I changed the nnU-Net code, I did the following:

> mkdir [YOUR_PATH_TO_NNU_NET_CODE]

> cd [YOUR_PATH_TO_NNU_NET_CODE]

> git add submodule https://github.com/MIC-DKFZ/nnUNet.git

> pip install -e [YOUR_PATH_TO_NNU_NET_CODE]

Set the environment variables that nnU-Net needs. I did that by adding them to my .bashrc:
```
export nnUNet_raw_data_base="/exports/lkeb-hpc/[YOUR_USER_NAME]/p16/nnUNet_raw_data_base" 
export nnUNet_preprocessed="/exports/lkeb-hpc/[YOUR_USER_NAME]/p16/nnUNet_preprocessed" 
export RESULTS_FOLDER="/exports/lkeb-hpc/[YOUR_USER_NAME]/p16/results"
```

Make sure those folders exist! 

## Set-up the p16 folder structure

Create a folder where the p16 data will be placed. For example:

> /exports/lkeb-hpc/[YOUR_USER_NAME]/p16/data

This folder will contain all the batches. For every batch, one only needs to create a folder batchX inside this folder and place a zip file BatchX.zip there (see procedure below). After running the scripts for that batch, a batch directory will look like this:

```
   /exports/lkeb-hpc/[YOUR_USER_NAME]/p16/data
   |
   |---- batch0
        | Batch0.zip            (only file manually placed)
        |
        |---- Batch0            (unzipped folder with DICOM scans)
        |
        |---- niftis            (niftis created from DICOM scans)
        |
        |---- segmentations     (automatic segmentations)
            |
            |----- TaskA        (segmentations created by TaskA)
            |....
        |---- corr              (medical expert segmentations)
                  
```


## Set-up variables used in continuous learning scripts

The continuous learning scripts use certain variables for paths that are often used or necessary for SLURM jobs. Make sure the paths exist! Put the variables inside the file ENVIRONMENT_VARIABLES.txt :

```
job_dir="[YOUR_JOB_DIR]" -- the SLURM job files are placed in this folder
log_dir="[YOUR_LOG_DIR]" -- the SLURM jobs logs are places here 
env_dir="/exports/lkeb-hpc/[YOUR_USER_NAME]/venv_environments/[YOUR_VENV_NAME]/bin/activate"
nnUNet_code_dir=[YOUR_PATH_TO_NNU_NET_CODE]
p16_dir="/exports/lkeb-hpc/[YOUR_USER_NAME]/p16/data" -- folder created in step 'Set-up the p16 folder structure'
```

## Create first P16 model

First, get all the p16 data used by the latest model of my thesis (batch0+batch1+batch2). To continue where my thesis ended, copy the contents of

> /exports/lkeb-hpc/smaijer/data/nnUNet_raw_data_base/nnUNet_raw_data/Task613

to

> $nnUNet_raw_data_base/Task500

This will be the first task of the P16 continuous learning cycle. nnU-Net will use this data to train a model. We do this with the following commands:

> source ENVIRONMENT_VARIABLES.txt

> source $env_dir

> python scripts/runTraining.py (make sure to enter "n" when asked if you want to use weights of previous model as the starting weights!)

# Continuous learning procedure

These are all the manual steps needed for 1 iteration of continuous learning:

1. Create a new folder batchX within the folder $p16_dir

2. Upload the zip file in the folder $p16_dir/batchX

3. Run ./0_createDataSetAndPerformInference.sh 
   - creates a new nnU-Net task
   - copies all the existing P16 scans to the new task
   - converts DICOMs to niftis and adds them to the new nnU-Net task
   - performs inference on the new scans using the most recent model
   - LOCATION AUTOMATED SEGMENTATIONS:
       $p16_dir/batchX/segmentations/TaskY
       (here, TaskY is the task of the most recent model)

4. Upload the following to the LUMC fileserver (you need to create the LUMC fileserver folders yourself):
   a. NIFTIS
   shark location:  

   > $p16_dir/batchX/niftis

   lumc fileserver:  

   > \\vf-mdlz-onderzoekstraject\mdlz-onderzoekstraject$\MRI - segmentaties\Batch[X]\scans

   b. AUTOMATED SEGMENTATIONS
   shark location:  

   > $p16_dir/batchX/segmentations/TaskY

   lumc fileserver:   
   
   > \\vf-mdlz-onderzoekstraject\mdlz-onderzoekstraject$\MRI - segmentaties\Batch[X]\segmentaties_auto
  
   c. KEY FILE

   shark location: 
   
   > $p16_dir/batchX/niftis/identify.txt

   lumc fileserver: 
   
   > \\vf-mdlz-onderzoekstraject\mdlz-onderzoekstraject$\MRI - segmentaties\Batch[X]

5. Wait for medical experts to correct the scans.

6. After manual correction, upload the corrected segmentations from:

   > \\vf-mdlz-onderzoekstraject\mdlz-onderzoekstraject$\MRI - segmentaties\BatchX\segmentaties_corr

  to:

  > $p16_dir/batchX/corr

7. Run ./1_evaluateCorrectionsAndStartNewTraining.sh
   - evaluates the predictions by comparing them to the corrections
   - copies the corrections to the task created by step 3
   - starts a training run for this new task (including preprocessing step)

# Tips

During performing step 0, niftis are created from the DICOMS. Please make sure to check identiy.txt to make sure there are no weird sequences - this can happen, as I hardcoded each T2 sequence that I don't want in createP16Dataset.py:

> if series_desc.startswith("T2") and not ("COR" in series_desc or "SPAIR" in series_desc or "SPIR" in series_desc):

