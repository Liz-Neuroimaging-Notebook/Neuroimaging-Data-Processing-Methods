# Difference in Gray Matter regions in Alzheimer's Disease

My workflow is the following:

| Given: **4 DICOM images of T1-weighted structural MRI.**    | 
| :---         |  
|              | 
|_Among the four DICOM images, two scans are from cognitively normal controls and the other two are from patiens with Alzheimer's disease (AD). It is hypothesized that multiple gray matter regions of the brain in AD present atrophy compared to healthy brains._  | 
| **Task: Assess the hypothesized GM atrophy in AD**   |
|Technique Used: **Voxel-based morphometry (VBM)**| 
 



CN= Control  
AD= Alzheimers Disease  

System Used: Mac OS Ventura 13.2.1

-----------------------------------------------------------------

| Step | VBM Workflow | Functionality |Package |
| ------------- | ------------- | ------------ |------------ |
| 1 | DICOM to NIFTI Conversion  |  Create .nii files            |   dcm2nii   |
| 2 | Bias Field Correction  | Corrects for differences in intensity               |  N4   |
| 3  |Brain Masking  | Used for brain extraction from skull, to separate non-brain tissue         |  ROBEX   |
| 4  | Linear Registration   |  Align brain images to a standard template, such as MNI            |  ANTS & Warp Image Multi Transform   |
| 5  | Tissue Segmentation  | Classify voxels as white matter, grey matter, or CSF             | FAST    |
| 6 | Smoothing "blurring" | Improve SNR (Signal-to-Noise-Ratio)             | FSLMaths -s     |
| 7 | VBM Difference Map  | Show where there are differences, such as GM, between groups. In this case a difference in GM concentration, using **probability maps**       |   FSLMaths -sub        |


-----------------------------------------------------------------
**To view DICOM file: **
```Powershell
/usr/local/afni-2011/dicom_hdr ADNI_002_S_0295_MR_MP-RAGE__br_raw_20060418193713091_1_S13408_I13722.dcm

```
<img width="500" alt="DICOM_files" src="https://github.com/Liz-Neuroimaging-Notebook/Neuroimaging-Data-Processing-Methods/assets/156251670/851a884b-58d1-4274-a95e-b12ca3cc327c">

-----------------------------------------------------------------
**Step 1: DICOM to NIFTI files**

-Create **output** folders for .json and .nii files

| PowerShell Command  | Folder |
| ------------- | ------------- |
| mkdir output1  | output1  |
| mkdir output2  | output2  |
| mkdir output3  | output3  |
| mkdir output4  | output4  |


**Use dcm2nii package**

| Controls      | Alzheimers Disease |
| ------------- | ------------- |
| /usr/local/mricron_03-2020/dcm2niix -o ./output1 ./S13408  | /usr/local/mricron_03-2020/dcm2niix -o ./output2 ./S15147  |
| /usr/local/mricron_03-2020/dcm2niix -o ./output4 ./S828320 | /usr/local/mricron_03-2020/dcm2niix -o ./output3 ./S795926  |



-----------------------------------------------------------------
**Step 2, 3 & 4: Bias Field, Brain Masking, and Linear Registration** using for-loop  
*each image (4 total) go through the for-loop. This way instead of doing each step for each image, we can perform 3 steps in one run. 
```Powershell
gedit for_loop_file.txt         #create a text file to store loop in
```

```Powershell
source /ifshome/hkim/.bash_profile

images=("CN_941_S_6254_t1.nii.gz" "S15147_MP-RAGE_20060601194452_2.nii" "S13408_MP-RAGE_20060418081744_3.nii" "S795926_Accelerated_Sagittal_IR-FSPGR_20190208101725_3.nii")
for img in "${images[@]}"; do

     # N4 Bias field correction
     N4BiasFieldCorrection -i $img -o [${img}_N4corrected.nii.gz,${img}_N4bias.nii.gz]
     
     # ROBEX for Brain Masking
     runROBEX.sh ${img}_N4corrected.nii.gz ${img}_Robex.nii.gz ${img}_Robex_mask.nii.gz

     # ANTS
     ANTS 3 --image-metric MI[./MNI_152_t1_masked.nii.gz,./${img}_Robex.nii.gz,1.0,32] - -do-rigid 0 --number-of-affine-iterations 1000x1000x1000 --number-of-iterations 0
     --use-rotation-header 1 --output-naming ./output/${img}_to_MNI152_T1

     #Warp Image Multi Transform 3
     WarpImageMultiTransform 3 ./${img}_Robex.nii ./output/${img}_to_MNI152_brain.nii.gz -R ./MNI_152_t1_masked.nii.gz ./output/${img}_to_MNI152_T1Affine.txt

done
```

```Powershell
source for_loop_file.txt     #to run loop
```

**Notes on ANTS:**
```Powershell
Example: ANTS 3 --image-metric MI[./MNI_152_t1_masked.nii.gz,./${img}_Robex.nii.gz,1.0,32]
--do-rigid 0 --number-of-affine-iterations 1000x1000x1000 --number-of-iterations 0
--use- rotation-header 1 --output-naming ./output/${img}_to_MNI152_T1

Quick explanation of variables in ANTS: 
   --do-rigid: this value can be either 1 (yes) or 0 (no). 0. Placed a zero since I am running an
   affine transformation. The extra parameters of shearing and scaling will help better infer
   differences between subject and template.
   
   --number-of-affine-iterations: for linear registration. We can use the default values, 1000x1000x1000
   
   --number-of-iterations: for nonlinear registration. Since I am preparing these images for VBM,
   I don’t need a non-linear registration here, thus, I placed a value of 0.
   
   --use-rotation-header: provides additional information about our matrix in our file.
   
   --output-naming: this is the new file name of our output file

```


<img width="615" alt="N4BiasField" src="https://github.com/Liz-Neuroimaging-Notebook/Neuroimaging-Data-Processing-Methods/assets/156251670/cd25f645-6e41-46cd-bb19-c60cc089122d">



2. ROBEX for brain masking
<img width="600" alt="Robex1" src="https://github.com/Liz-Neuroimaging-Notebook/Neuroimaging-Data-Processing-Methods/assets/156251670/626be3b8-fdcf-4826-935a-da4782a2a4ec">

<img width="600" alt="Robex2" src="https://github.com/Liz-Neuroimaging-Notebook/Neuroimaging-Data-Processing-Methods/assets/156251670/df63610e-8fa4-4ed2-acca-7bc758228623">


<img width="600" alt="AntsandWarp" src="https://github.com/Liz-Neuroimaging-Notebook/Neuroimaging-Data-Processing-Methods/assets/156251670/58a64c2b-5138-46e6-80b8-7d4c91b5873e">





-----------------------------------------------------------------
**Step 5: Tissue Segmentation**  
We use FAST to segment our images.

```Powershell
  -n,--class     number of tissue-type classes; default=3
  -H,--Hyper     segmentation spatial smoothness; default=0.1
  -l,--lowpass   bias field smoothing extent (FWHM) in mm; default=20
```

```Powershell
fast -n 3 -t 1 -H 0.1 -l 20 -o ./CN_941_to_MNI152_brain ./CN_941_S_6254_t1.nii.gz_to_MNI152_b rain.nii.gz
fast -n 3 -t 1 -H 0.1 -l 20 -o ./S15147_to_MNI152_brain ./S15147_MP-RAGE_20060601194452_2.nii_to_MNI152_brain.nii.gz
fast -n 3 -t 1 -H 0.1 -l 20 -o ./S13408_to_MNI152_brain ./S13408_MP-RAGE_20060418081744_3.nii_to_MNI152_brain.nii.gz
fast -n 3 -t 1 -H 0.1 -l 20 -o ./S795926_to_MNI152_brain ./S795926_Accelerated_Sagittal_IR-FSPGR_20190208101725_3.nii_to_MNI152_brain.nii.gz
```

```Powershell  
Example of outputs for ONE file when using FAST (6 files total each): 
     CN_941_to_MNI152_brain_seg.nii.gz
     CN_941_to_MNI152_brain_pve_O.nii.gz
     CN_941_to_MNI152_brain_pve_1.nii.gz 
     CN_941_to_MNI152_brain_pve_2.nii.gz 
     CN_941_to_MNI152_brain_pveseg.nii.gz
     CN_941_to_MI152_brain_mixeltype.nii.gz
```

<img width="600" alt="Tissue_Segmentation_" src="https://github.com/Liz-Neuroimaging-Notebook/Neuroimaging-Data-Processing-Methods/assets/156251670/d337d3fe-8c4c-4234-843d-2a03930b9af1">


-----------------------------------------------------------------
**Step 6 : Smoothing**  
*apply Gaussian smoothing for final measurement map (GM probability map) with kernel size of 5mm.  
Use FSLMaths -s

```Powershell
fslmaths ./CN_941_to_MNI152_brain_pve_1.nii.gz -s 5 ./CN_941_to_MNI152_pve_1_blur.nii.gz
fslmaths ./S15147_to_MNI152_brain_pve_1.nii.gz -s 5 ./S15147_to_MNI152_pve_1_blur.nii.gz
fslmaths ./S13408_to_MNI152_brain_pve_1.nii.gz -s 5 ./S13408_to_MNI152_pve_1_blur.nii.gz
fslmaths ./S795926_to_MNI152_brain_pve_1.nii.gz -s 5 ./S795926_to_MNI15_pve_1_blur.nii.gz
```
```Powershell
Notice how we use **_brain_pve_1.nii.gz**
This file corresponds to grey matter segmentation.
```
```Powershell
Output for all Control and AD images (4 total):
    MI_152_t1_pve_1_blur.nii.gz
    CN_941_to_MNI152_pve_1_blur.nii.gz
    S15147_to_MNI152_pve_1_blur.nii.gz
    S13408_to_MNI152_pve_1_blur.nii.gz
    S795926_to_MI15_pve_1_blur.nii.gz
```
```Powershell
I copied MNI_152_t1_pve_1blur.nii.gz from another folder (MNI is our template)
   • cp -rf MNI_152_t1_pve_1_blur.nii.gz ~/[insert path to copy to]
```
**Average the smoothed maps within each group (Controls & AD):**

```Powershell
Controls:
fslmaths CN_941_to_MNI152_pve_1_blur.nii.gz -add S13408_to_MNI152_pve_1_blur.nii.gz -div 2 control_group_average.nii.gz
```

```Powershell
Alzheimers Disease:
fslmaths S15147_to_MNI152_pve_1_blur.nii.gz -add S795926_to_MNI15_pve_1_blur.nii.gz -div 2 AD_group_average.nii.gz
```
<img width="600" alt="Smoothing_blurring" src="https://github.com/Liz-Neuroimaging-Notebook/Neuroimaging-Data-Processing-Methods/assets/156251670/ea950751-7445-43f0-821f-5dd626d57f2a">

<img width="600" alt="VBM_AD Control" src="https://github.com/Liz-Neuroimaging-Notebook/Neuroimaging-Data-Processing-Methods/assets/156251670/fd94791f-e5e6-4402-99f3-9bf9a0d8e984">

--------------------------------------------------------------------  
**Step 7: VBM Difference Map**  
We use FSLMaths -sub 

**Practice Exercise- VBM Difference Map between 4 images and MNI Template **
```Powershell
   fslmaths ./CN_941_to_MNI152_pve_1_blur.nii.gz -sub ./MNI_152_t1_pve_1_blur.nii.gz ./CN_941_to_MNI152_Regular_VBM_Subtraction.nii.gz
   fslmaths ./S15147_to_MNI152_pve_1_blur.nii.gz -sub ./MNI_152_t1_pve_1_blur.nii.gz ./S15147_to_MNI152_Regular_VBM_Subtraction.nii.gz
   fslmaths ./S13408_to_MNI152_pve_1_blur.nii.gz -sub ./MNI_152_t1_pve_1_blur.nii.gz ./S13408_to_MNI152_Regular_VBM_Subtraction.nii.gz
   fslmaths S795926_to_MNI15_pve_1_blur.nii.gz -sub ./MNI_152_t1_pve_1_blur.nii.gz ./S795926_to_MNI15_Regular_VBM_Subtraction.nii.gz
```
<img width="623" alt="6_a_VBM Difference Map" src="https://github.com/Liz-Neuroimaging-Notebook/Neuroimaging-Data-Processing-Methods/assets/156251670/ee07f175-8d92-4784-bba5-b7d1a763196c">


<img width="600" alt="6 bVBM_Difference_Map" src="https://github.com/Liz-Neuroimaging-Notebook/Neuroimaging-Data-Processing-Methods/assets/156251670/fd5d5988-ef91-4af2-a0b3-c53d82dffd8e">

**VBM Difference Map we want between Controls & AD images:**  
Subtract the average map of AD from that of controls. This will let us see where atrophy has occurred. 
```Powershell
fslmaths control_group_average.nii.gz -sub AD_group_average.nii.gz ControlAD_Regular_VBM_Subtracted_map.nii.gz
```


**View difference map using FSL View:**
```Powershell
fslview ./output/ControlAD_Regular_VBM_Subtracted_map.nii.gz MNI_152_t1_masked.nii.gz
```

<img width="600" alt="Final_VBM_AD Control_1" src="https://github.com/Liz-Neuroimaging-Notebook/Neuroimaging-Data-Processing-Methods/assets/156251670/cae36df3-c397-4d4e-b675-18779538de6f">

<img width="639" alt="Final_VBM_AD Control_2" src="https://github.com/Liz-Neuroimaging-Notebook/Neuroimaging-Data-Processing-Methods/assets/156251670/47773b24-148b-4c1e-9e21-8ff56f07ba4c">


Tip: Make sure to press tab when adding in files to a path. This makes sure your file is typed out exactly how it is in the folder. 



# Conclusion 
**Areas with most atrophy:**  

<img width="460" alt="Hippocampus_atrophy" src="https://github.com/Liz-Neuroimaging-Notebook/Neuroimaging-Data-Processing-Methods/assets/156251670/f12d69ef-6c37-43bc-9e9d-34476feb4b83">  


<img width="460" alt="Parietal_lobe_atrophy" src="https://github.com/Liz-Neuroimaging-Notebook/Neuroimaging-Data-Processing-Methods/assets/156251670/72dceee5-40ba-4384-a3fd-ed9dcc146fe8">  

<img width="460" alt="Thalamus_atrophy" src="https://github.com/Liz-Neuroimaging-Notebook/Neuroimaging-Data-Processing-Methods/assets/156251670/2d27d5d3-2126-4456-bfeb-35467d67e370"> 

Although regions such as the hippocampus and parahippocampal gyrus are considered the most predictive structural brain biomarkers for Alzheimer's (AD), neuroimaging studies are showing how parietal hypoperfusion could also be a biomarker for AD (Jacobs et al., 2012). Parietal hypoperfusion in AD can start as early as the prodromal stage of AD, a time when we start to see mild cognitive impairment in AD patients (Wu et al., 2021). Parietal hypoperfusion is linked to grey matter loss, and it is grey matter loss in the parietal lobe which we see in our AD images above. Grey matter loss in this area can lead to spatial and navigational impairments, apraxia, or visuospatial processing deficits. While grey matter loss in the hippocampus aligns with the loss of long-term memory consolidation often seen in AD patients. GM loss in the thalamus can also lead to sensory processing impairments, as well as movement abnormalities. With the latter making sense, since the thalamus is a major relay center for sensory and movement information.

**References:**
Jacobs, H. I., Van Boxtel, M. P., Jolles, J., Verhey, F. R., & Uylings, H. B. (2012). Parietal cortex matters in Alzheimer's disease: an overview of structural, functional and metabolic findings. Neuroscience and biobehavioral reviews, 36(1), 297–309. 

Wu, Z., Peng, Y., Hong, M., & Zhang, Y. (2021). Gray Matter Deterioration Pattern During Alzheimer's Disease Progression: A Regions-of-Interest Based Surface Morphometry Study. Frontiers in aging neuroscience, 13, 593898. 




