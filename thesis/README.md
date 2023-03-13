This folder contains shell scripts to create job files for various nnUnet commands.

The most basic pipeline consists out of the following steps:

1. Preprocessing. 
2. Training.
3. Postprocessing.
4. Prediction.
5. Evaluation.

The following-shell script performs all those steps:

`` run_all.sh ``

Following job file is an example SLURM job file containing all the steps:

`` job_file_all_steps.job `` 

## Preprocessing

shell-script:

`` run_preprocess.sh``

nnU-Net command:

`` nnUNet_plan_and_preprocess ``


goal: extracts mean dataset properties to resample all scans to same voxel spacings, crop to non-zero output, and if applicable, also creates downsampled versions of the scans to use in multi-phase model. Outputs get put at location of nnU-Net variable of $nnUNet_preprocessed

## Training

shell-script:

`` run_train.sh (single fold)``
`` run_train_all.sh (5 folds)``

nnU-Net command:

`` nnUNet_train ``

goal: trains a single fold and performs inference on the validation set of that fold at the end of training.

## Postprocessing

shell-script:

`` run_determine_postprocessing.sh ``

nnU-Net command:

`` nnUNet_determine_postprocessing ``

goal: trains a single fold and performs inference on the validation set of that fold at the end of training.

## Prediction

shell-script:

`` run_inference.sh ``

nnU-Net command:

`` nnUNet_predict ``

goal: predicts all the scans at a given location.

## Evaluation

shell-script:

`` run_evaluate_folder.sh ``

nnU-Net command:

`` nnUNet_evaluate_folder ``

goal: compares the ground truths with the prediction and calculates various metrics.