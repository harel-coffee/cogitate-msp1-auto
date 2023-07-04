# Introduction

Here we provide an overview of the preprocessing and analysis pipeline for the fMRI modality, as well as a description and instructions to use the provided code.

**Authors:** *Yamil Vidal, David Richter, Aya Khalaf*

The code for different preprocessing steps and analyses are provided in different folders:

-	bidscoin: primarily contains the custom yaml map mapping the raw dicom data to BIDS converted nifti data on the project (MPI) HPC. For more information on bidscoin see: [bidscoin.readthedocs.io](bidscoin.readthedocs.io)
-	data_rejection: scripts to assess data quality and flag datasets/participants to be rejected from analysis.
-	dicom2bids: script that runs the dicom to BIDS converter.
-	glm: scripts to (1) extract confound regressors from fmriprep and (2) run/submit 1st (run) level GLMs, as well as 2nd (subject) and 3rd (group) level GLMs using FSL FEAT. Also contains subfolder with fsf templates used by FEAT.
-	logfiles_and_checks: scripts to (1) create events.tsv files from experiment log files per experiment (exp1, exp2, evcLoc). The resulting events.tsv + assocaited .json sidecar are BIDS compliant. Also performs various sanity checks on MRI data and logs. Another set of scripts (2) creates regressor event txt files as required by MRI analysis software (e.g. FSL FEAT; 3 column format).
-	masks: script to create (anatomical) ROI masks.
-	putative_ncc: scripts to run putative NCC analysis (assumes that FSL FEAT outputs already exist; see scripts in ./glm folder).
-	decoding: scripts to (1) obtain single trial estimates, (2) run subject level and group level searchlight and ROI decoding analyses, (3) plot group level searchlight accuracy maps on axial brain slices and brain surfaces, and (4) plot group level ROI accuracies on brain surfaces.    
-	gppi: scripts to (1) convert 4D nifiti files to 3D nifti files, (2) smooth 3D nifti files, (3) run subject level GLMs, (4) run subject level and group level gppi analyses, and (5) plot group level gppi stats maps on axial brain slices and brain surfaces.

# fMRI data pipeline overview

## Preprocessing
0. Setup dicom to BIDS converter (once); BIDSMAPPER
1. Conversion of MRI DICOM data to BIDS; BIDSCOINER
2. Creation of events.tsv files; PYTHON CODE
3. BIDS validation; BIDSVALIDATOR
4. MRI data quality checks; MRIQC
5. Data rejection; PYTHON CODE
6. MRI preprocessing & visual data/preprocessing quality checks; FMRIPREP

## Analysis
7. Create (anatomical) ROI masks
8. Create regressor event txt files (3 column format)
9. Run 1st, 2nd and 3rd level GLMs
10. Create Decoding ROIs
11. Create seeds for GPPI analysis
12. Run Putative NCC analysis
13. Putative NCC analysis: Generate data for tables
14. Putative NCC analysis: Generate figures
15. Decoding Analyses
16. gppi Analyses

# fMRI data pipeline details

0. BIDS COIN SETUP
Setup of data processing: Run only once during setup (requires sample dataset)

    module purge
    module load bidscoin/3.6.3
    bidsmapper /COGITATE/fMRI/processed/temp_raw_for_bidscoin /COGITATE/fMRI/processed/bids


1. DICOM TO BIDS
Converts DICOM to BIDS compliant niftis

    module purge
    module load bidscoin/3.6.3
    cd /COGITATE/fMRI/processed/bids/code/dicom_to_bids
    python 01_convert_dicom_to_bids.py


2. Create `events.tsv` files & perform MRI log file checks (custom code) ###
Create `events.tsv` files per run from experiment native log files

    module purge
	module load Python/3.8.6-GCCcore-10.2.0
	cd /COGITATE/fMRI/processed/bids/code/logfiles_and_checks
	python 01_exp1_create_events_tsv_file.py
	python 01_evcLoc_create_events_tsv_file.py


3. BIDS Validator (https://neuroimaging-core-docs.readthedocs.io)
Validate BIDS compliance of dataset

	module purge
	module load nodejs/12.16.1-GCCcore-7.3.0 
	cd /COGITATE/fMRI/processed
	bids-validator bids


4. a) MRI QC (https://mriqc.readthedocs.io; https://github.com/marcelzwiers/mriqc_sub)
Run MRI QC for visual inspection of (f)MRI data quality. Perform visual inspection of each runs data (see ./bids/derivatives/mriqc).

	module purge
	module load mriqc
	cd /COGITATE/fMRI/processed
	mriqc_sub.py /COGITATE/fMRI/processed/bids -t 48 -w /COGITATE/fMRI/processed/scratch/mriqc_workdir -o /COGITATE/fMRI/processed/bids/derivatives


4. b) MRI QC group level
Run MRI QC at the group level

	module purge
	module load mriqc
	cd /COGITATE/fMRI/processed
	mriqc_group.py bids


5. Data rejection using MRI QC IQMs (custom code)
Extract IQMs of interest from MRI QC and reject runs/participants from further analysis (run only AFTER all data has been processed with MRIQC)

	module purge
	module load Python/3.8.6-GCCcore-10.2.0
	cd /COGITATE/fMRI/processed/bids/code/data_rejection
	python 01_analyze_MRIQC_IQMs.py


6. fMRIprep. (https://fmriprep.org; https://github.com/marcelzwiers/fmriprep_sub)
Preprocess (f)MRI data. Perform visual inspection of each runs data (see ./bids/derivatives/fmriprep)

	module purge
	module load fmriprep/20.2.3
	cd /COGITATE/fMRI/processed
	fmriprep_sub.py /COGITATE/fMRI/processed/bids -w /COGITATE/fMRI/processed/scratch/fmriprep_workdir --time 80 --mem_mb 30000 -n 6 -a " --ignore sbref slicetiming --output-spaces T1w MNI152NLin2009cAsym"


7. a) Create anatomical ROI masks (custom code)
Besides running this code for the desired participants, it should be also run for the *MNI152NLin2009cAsym* standard brain (for group lvl analyses).
For this we used the precomputed FreeSurfer output that can be found here: https://figshare.com/articles/dataset/FreeSurfer_reconstruction_of_the_MNI152_ICBM2009c_asymmetrical_non-linear_atlas/4223811

	module purge
	module load Python/3.8.6-GCCcore-10.2.0
	module load FreeSurfer
	module load FSL
	cd /COGITATE/fMRI/processed/bids/code/masks
	python 01_create_ROI_masks.py


7. b) Resample anatomical ROI masks to target space
The standard space used is MNI152NLin2009cAsym

	module purge
	module load ANTs
	cd /COGITATE/fMRI/processed/bids/code/masks
	python 02_resample_ROI_masks_to_target_space.py


7. c) Combine ROIs to create theory specific ROIs
This also creates FFA and LOC masks used for the creation of GPPI seeds

	module purge
	cd /COGITATE/fMRI/processed/bids/code/masks
	python 03_create_theory_ROI_masks.py


7. d) Resample group level anatomical ROI masks to target space
The standard space used is MNI152NLin2009cAsym

	cd /COGITATE/fMRI/processed/bids/code/masks
	bash 04_resample_MNI152_ROIs.sh


7. e) Combine ROIs to create theory specific ROIs (group level)

	cd /COGITATE/fMRI/processed/bids/code/masks
	python 05_create_theory_ROI_masks_MNI152.py


8. Create regressor event txt file (custom code)
Create regressor event txt files; 3 column format; 1 per regressors (FSL FEAT compliant) from information in `events.tsv` files per run.
 
	module purge
	module load Python/3.8.6-GCCcore-10.2.0
	cd /COGITATE/fMRI/processed/bids/code/logfiles_and_checks
	python 02_exp1_create_regressor_txt_files.py
	python 02_evcLoc_create_regressor_txt_files.py


9. Run 1st, 2nd and 3rd level GLMs
Create confound regressor files to be used in 1st level GLMs. Then run first and second level GLMs using FSL FEAT; adjust analysis level in python script. Perform visual inspection of each run's output data (see ./bids/derivatives/fslFeat).

	module purge
	module load Python/3.8.6-GCCcore-10.2.0
	module load Spyder/4.1.5-foss-2019a-Python-3.7.2
	module load FSL
	cd /COGITATE/fMRI/processed/bids/code/glm
	python 01_create_confound_regressor_ev_file.py
	python 02_run_fsf_feat_analyses.py    # Do for each GLM lvl


0. Create Decoding ROIs
Requires anatomical ROIs (step 7) and GLMs (step 9).

	module purge
	module load Python/3.8.6-GCCcore-10.2.0
	cd /COGITATE/fMRI/processed/bids/code/decoding_rois
	python 01_create_decoding_rois_all_runs.py
	python 02_create_decoding_rois_leave_one_run_out.py


11. Create seeds for GPPI analysis
Requires anatomical ROIs (step 7) and GLMs (step 9).

	module purge
	module load Python/3.8.6-GCCcore-10.2.0
	cd /COGITATE/fMRI/processed/bids/code/seeds_for_gppi
	python 01_create_gppi_seeds.py


12. Run Putative NCC analysis
Run putative NCC analysis on FSL Feat outputs.

	module purge
	module load Python/3.8.6-GCCcore-10.2.0
	module load FSL
	cd /COGITATE/fMRI/processed/bids/code/putative_ncc

	python 01_PutativeNCC_analysis_on_FEAT_copes.py                  # Group lvl univariate
	python 02_putative_ncc_create_C_not_A_or_B_maps.py               # Exclude voxels responsive to task goals and task relevance (Group lvl univariate)

	python 03_putative_ncc_analysis_on_FEAT_copes_subject_level.py   # Subject lvl univariate
	python 04_putative_ncc_subject_level_create_C_not_A_or_B_maps.py # Exclude voxels responsive to task goals and task relevance (Subject lvl univariate)

	python 05_multivariate_putative_ncc_analysis.py                  # Group lvl multivariate
	python 06_multivariate_putative_ncc_create_C_not_A_or_B_maps.py  # Exclude voxels responsive to task goals and task relevance (Group lvl multivariate)

	python 07_putative_ncc_merge_phases.py                           # Combine optimization and replication phases, for plotting purposes


13. Putative NCC analysis: Generate data for tables
Count detected voxels in each anatomical ROI and save data to csv files (used to produce tables).
Requires anatomical masks (7) and the results of putative NCC analizes (12).

	module purge
	module load Python/3.8.6-GCCcore-10.2.0
	cd /COGITATE/fMRI/processed/bids/code/putative_ncc_tables
	python 01_putative_ncc_group_level_tables.py
	python 02_putative_ncc_subject_level_tables.py
	python 03_multivariate_putative_ncc_group_level_tables.py


14. Putative NCC analysis: Generate figures
Requires Slice Display (https://github.com/bramzandbelt/slice_display), and MATLAB

	module purge
	module load Python/3.8.6-GCCcore-10.2.0
	cd /COGITATE/fMRI/processed/bids/code/putative_ncc_plotting
	Putative_NCC_01_univariate.m    # Univariate pNCC, main figure (5) and individual stimulus categories
	Putative_NCC_02_AB.m            # Areas responsive to task goals and task relevance
	Putative_NCC_03_multivariate.m  # Multivariate pNCC
	Putative_NCC_04_z_maps.m        # Individual z maps for each stimulus category and condition (relevant and irrelevant)

15. Decoding Analysis
**a)** Obtain single trial estimates which are required for all the rest of the decoding analyzes.

	module load nibetaseries/0.6.0
	singularity run --cleanenv -B /mnt/beegfs:/mnt/beegfs -B /hpc:/hpc ${NIBETASERIES_SIMG} nibs -c trans_x trans_x_derivative1  trans_x_power2 trans_x_derivative1_power2 trans_y trans_y_derivative1 trans_y_power2 trans_y_derivative1_power2 trans_z trans_z_derivative1 trans_z_power2 trans_z_derivative1_power2 rot_x rot_x_derivative1 rot_x_power2 rot_x_derivative1_power2 rot_y rot_y_derivative1 rot_y_power2 rot_y_derivative1_power2 rot_z rot_z_derivative1 rot_z_power2 rot_z_derivative1_power2 csf white_matter --participant-label PARTICIPANT_LABEL  --session-label V1 --nthreads 32  --normalize-betas --estimator lss --hrf-model 'spm'  -w /mnt/beegfs/XNAT/COGITATE/fMRI/phase_2/processed/bids/derivatives/betaseries  /mnt/beegfs/XNAT/COGITATE/fMRI/phase_2/processed/bids fmriprep /mnt/beegfs/XNAT/COGITATE/fMRI/phase_2/processed/bids/derivatives/ participant

**b)** Run searchlight decoding
Subject level category decoding

	module purge
	module load Anaconda3/2020.11
	source /hpc/shared/EasyBuild/apps/Anaconda3/2020.11/bin/activate
	conda activate /hpc/users/$USER/.conda/envs/mne_ecog01
	cd /COGITATE/fMRI/processed/bids/code/decoding
	python searchlight_category_decoding_subject_level.py  

Subject level orientation decoding

	module purge
	module load Anaconda3/2020.11
	source /hpc/shared/EasyBuild/apps/Anaconda3/2020.11/bin/activate
	conda activate /hpc/users/$USER/.conda/envs/mne_ecog01
	cd /COGITATE/fMRI/processed/bids/code/decoding
	python searchlight_orientation_decoding_subject_level.py

Subject level stim vs baseline decoding to be used as input for multivariate putative NCC analysis

	module purge
	module load Anaconda3/2020.11
	source /hpc/shared/EasyBuild/apps/Anaconda3/2020.11/bin/activate
	conda activate /hpc/users/$USER/.conda/envs/mne_ecog01
	cd /COGITATE/fMRI/processed/bids/code/decoding
	python searchlight_stim_baseline_decoding_subject_level.py

Group level analysis 

	module purge
	module load Anaconda3/2020.11
	source /hpc/shared/EasyBuild/apps/Anaconda3/2020.11/bin/activate
	conda activate /hpc/users/$USER/.conda/envs/mne_ecog01
	cd /COGITATE/fMRI/processed/bids/code/decoding
	python searchlight_decoding_group_analysis.py                            # Plots group level accuracy maps on axial brain slices 

**c)** Obtain searchlight decoding tables
Searchlight tables based on the group level searchlight accuracy maps  

	module purge
	module load Anaconda3/2020.11
	source /hpc/shared/EasyBuild/apps/Anaconda3/2020.11/bin/activate
	conda activate /hpc/users/$USER/.conda/envs/mne_ecog01
	cd /COGITATE/fMRI/processed/bids/code/decoding
	python searchlight_group_level_tables.py                         

**d)** Run ROI decoding
Subject level category decoding

	module purge
	module load Anaconda3/2020.11
	source /hpc/shared/EasyBuild/apps/Anaconda3/2020.11/bin/activate
	conda activate /hpc/users/$USER/.conda/envs/mne_ecog01
	cd /COGITATE/fMRI/processed/bids/code/decoding
	python roi_category_decoding_subject_level.py  

Subject level orientation decoding

	module purge
	module load Anaconda3/2020.11
	source /hpc/shared/EasyBuild/apps/Anaconda3/2020.11/bin/activate
	conda activate /hpc/users/$USER/.conda/envs/mne_ecog01
	cd /COGITATE/fMRI/processed/bids/code/decoding
	python roi_orientation_decoding_subject_level.py 

Group level analysis 

	module purge
	module load Anaconda3/2020.11
	source /hpc/shared/EasyBuild/apps/Anaconda3/2020.11/bin/activate
	conda activate /hpc/users/$USER/.conda/envs/mne_ecog01
	cd /COGITATE/fMRI/processed/bids/code/decoding
	python roi_decoding_group_analysis.py                           

**d)** Test IIT decoding predictions
Subject level category decoding evaluating accuracies obtained with IIT ROIs only vs accuracies obtained with IIT+PFC ROIs

	module purge
	module load Anaconda3/2020.11
	source /hpc/shared/EasyBuild/apps/Anaconda3/2020.11/bin/activate
	conda activate /hpc/users/$USER/.conda/envs/mne_ecog01
	cd /COGITATE/fMRI/processed/bids/code/decoding
	python roi_category_decoding_testing_IIT_predictions_combined_features.py 

Group level analysis to determine if the difference between IIT+PFC accuracies and IIT accuracies is statistically significant  

	module purge
	module load Anaconda3/2020.11
	source /hpc/shared/EasyBuild/apps/Anaconda3/2020.11/bin/activate
	conda activate /hpc/users/$USER/.conda/envs/mne_ecog01
	cd /COGITATE/fMRI/processed/bids/code/decoding
	python roi_decoding_group_analysis_IIT_predictions.py   

**e)** Plot searchlight and ROI decoding results on a brain surface
Plot group level searchlight decoding accuracy maps on a brain surface 

	module purge
	module load Anaconda3/2020.11
	source /hpc/shared/EasyBuild/apps/Anaconda3/2020.11/bin/activate
	conda activate /hpc/users/$USER/.conda/envs/mne_ecog01
	cd /COGITATE/fMRI/processed/bids/code/decoding
	python MSP1_searchlight_decoding_plots.py 

Plot group level ROI accuracies on a brain surface 

	module purge
	module load Anaconda3/2020.11
	source /hpc/shared/EasyBuild/apps/Anaconda3/2020.11/bin/activate
	conda activate /hpc/users/$USER/.conda/envs/mne_ecog01
	cd /COGITATE/fMRI/processed/bids/code/decoding
	python MSP1_roi_decoding_plots.py                                   

16. gppi Analysis
Requires MATLAB and SPM

**a)** Apply 3D nifti conversion and smoothing

	cd /COGITATE/fMRI/processed/bids/code/gppi
	nifti3D_conversion.m      # Convert 4D niftis to 3D niftis for SPM analaysis
	smoothing.m               # Smooth 3D niftis

**b)** Run gppi on combined conditions

	cd /COGITATE/fMRI/processed/bids/code/gppi
	glm_subject_level_combined.m           # Subject level GLM with category regressors collapsed across the relevant and irrelevant conditions (i.e face, object, letter, and false font regressors) 
	gppi_subject_level_combined.m          # Subject level gppi analysis

Group level gppi analysis

	module purge
	module load Anaconda3/2020.11
	source /hpc/shared/EasyBuild/apps/Anaconda3/2020.11/bin/activate
	conda activate /hpc/users/$USER/.conda/envs/mne_ecog01
	cd /COGITATE/fMRI/processed/bids/code/gppi
	python gppi_group_analysis.py        # Plots group level stats map on axial brian slices 

**c)** Run gppi on separate conditions

	cd /COGITATE/fMRI/processed/bids/code/gppi
	glm_subject_level.m                                    # Subject level GLM with separate category regressors for the relevant and irrelevant conditions (i.e relevant_face, relevant_object, relevant_letter, and relevant_false font) 
	gppi_subject_level.m                                   # Subject level gppi analysis

Group level gppi analysis

	module purge
	module load Anaconda3/2020.11
	source /hpc/shared/EasyBuild/apps/Anaconda3/2020.11/bin/activate
	conda activate /hpc/users/$USER/.conda/envs/mne_ecog01
	cd /COGITATE/fMRI/processed/bids/code/gppi
	python gppi_group_analysis.py        # Plots group level stats map on axial brian slices 

**d)** Obtain gppi tables

	module purge
	module load Anaconda3/2020.11
	source /hpc/shared/EasyBuild/apps/Anaconda3/2020.11/bin/activate
	conda activate /hpc/users/$USER/.conda/envs/mne_ecog01
	cd /COGITATE/fMRI/processed/bids/code/gppi
	python gppi_group_level_tables.py    # gppi tables obtained based on the group level analysis 

**e)** Plot gppi analysis results on a brain surface

	cd /COGITATE/fMRI/processed/bids/code/gppi
	MSP1_gppi_plots.py            # Plot group level gppi stats map on a brain surface