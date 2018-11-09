# CBS tools unofficial doc

## Install
* Install MIPAV: version 7.0.1 should work fine for CBS tools version 3.0.8
* Install the CBS tools plugins:
* Download from http://www.nitrc.org/projects/cbs-tools/
* Plugins → Install plugin
* Select the .jar files to install
* Click Install Plugin(s)
* Make sure you have also installed:
  * The JIST-CRUISE plugin
  * The external-scripts plugin

For more information on install, check the [user guide](https://www.cbs.mpg.de/169392/cbs-tools-userguide.pdf)

Note: some of the CBS tools (non linear co-registration, surface coregistraion) call bash scripts and have [ANTS](http://stnava.github.io/ANTs/) as dependencies and might therefore not work on non UNIX operating systems. It also requires some extra steps to specify the location of the ANTS bash scripts.

Check memory usage: 24 Gb minimum
Allocate more if necessary: Help → Memory allocation
Will require to restart MIPAV

Start the JIST layout tools
Plugins → JIST → JIST layout tools

The first time you do this you will have to:
* Set the directory used for storing the JIST module definitions
* Rebuild the libraries: Project → Rebuild library
* Set some Global Preferences: Project → Global Preferences
  * Preferred data format: LayoutXML
  * Preferred compression: 'gzip' (‘bzip2 if you are space greedy)
  * Debug level: 0 or 1 (higher values make JIST more verbose)

Notes:
* MIPAV and FSL are fine reading .nii.gz files directly but not SPM. You must first gunzip the files using the gunzip matlab command.
* Only volumes (.nii) will be compressed. Surface files (`*.vtk`) will not.


## nighres
Python wrappers for the CBS tools are available (no installation of MIPAV is required). Allows integration to nipype (???).
* [nighres](http://nighres.readthedocs.io/en/latest/)
* [article](https://riojournal.com/article/12346/)

For more check Pilou’s and Julia’s github repos
https://github.com/piloubazin
https://github.com/juhuntenburg

For more MIPAV / CBS tools layout (including high-res surface coregistration)
The code of Julia’s paper (https://doi.org/10.1093/cercor/bhx030) is there: https://github.com/juhuntenburg/myelinconnect


## IMPORTANT to know when using SPM and MIPAV
s-form and q-form transformation matrices
Nifti image headers have 2 transformation matrices. It is possible to independently report or modify the qform or sform fields.

The qform and sform stored in a nifti file are intended to fulfil the following functions:
* specify the handedness of the coordinate system (important for getting left and right correct)
* specify the original scanner coordinates (qform only)
* specify standard space coordinates (sform only)
* specify a relationship with another image's coordinates (sform only)

From: https://nifti.nimh.nih.gov/nifti-1/documentation/nifti1fields/nifti1fields_pages/qsform_brief_usage/document_view

MIPAV works on the qform and SPM on the sform. So you might run into problems because of that.

More info here:
http://gru.stanford.edu/doku.php/mrTools/coordinateTransforms
https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/Orientation%20Explained


## Reading VTK files
The vtk surfaces from the CBS tools have 4 parts:
* Header
* Vertices: that parts stays the same between inflated and non inflafed surfaces)
* Faces
* Mapping: X values (one column / layer) for each vertex (each row is a vertex).

Extracting data from surfaces
`*.vtk` files are text files that can be easily read in simple text editor. I have created a set of matlab function to read, extract data, threshold and write back the data from those surfaces. The gifti function from SPM should do the trick. Otherwise I also wrote write_vtk.m and read_vtk.m files


## Visualizing surfaces
Use Paraview: http://www.paraview.org/


## Drawing ROI on surfaces
Use the modified version of Cosmetic


## Using the CBS-tools
Open one of the segmentation layout from: `MipavRootFolder/plugins/layouts-r3.0`

For example: `brain-segmentation-MP2RAGE-7T-MNI-space.LayoutXML`

Several subjects can be processed in parallel pipelines, but the data must entered in the same order in each input module.

If there is only one file in a given module then this file will be used for all subject (e.g. atlas, template…) in parallel pipelines.

If there is only one file in a given module then this file will be used for all subject (e.g. atlas, template…)
mipav/plugins/atlases/MNI-template/MNI152_T1_04mm_brain.nii.gz
mipav/plugins/atlases/brain-segmentation-prior3.0/brain-atlas-3.0.txt

Modules have a number of obligatory inputs: top white pins

The top grey pins are either:
* for optional inputs
* or parameters that can be read from a file
* or passed on from another module
* or set manually in the parameter window

The bottom pins are for the output management

Modules can easily be linked to one another

Modules can also also renamed.

Save the layout and start the process manager:
Project → Process manager

The whole pipeline can be run at once Scheduler -> Start Scheduler

Each process with a “ready” status, can be started individually (“run” button or Right click → run)

The priority of each process can be modified (Right click → Promote/Demote)

Each parallel pipeline (e.g. subject) gets a different number (0000, 0001, 0002, …)

The letters are related to the parent process of the selected process.

The data of every step will be saved in the output directory specified in the layout preference and organized as follow:
`OutputDir/PipelineNumber/Process-ID/ModuleName/`

If a layout has been modified (change of parameters, new step added, some output files have been moved) some process (and sometimes all of its children processes) might have an “out of sync” status.

This can be fixed process by process (Right click → Clean) or for all the incriminated processes (Scheduler → Clean Out of Synch).

Sometimes a process might fail to complete… How surprising!

To help you debug what could have gone wrong , check the debug.err and debug.out files in folder of that process

One of the usual suspect is “Not enough memory”, so either allocate more RAM or run fewer processes at once.

Other classical issues:
* the input file has been moved
* you don’t have write access to the output folder
* your hard drive is full

Once you have fixed the issue you can clean and re-run the process.


## Brain segmentation
`brain-segmentation-*.LayoutXML`

There is one pipeline optimised for the MP2RAGE of the 7T at the MPI CBS.

Will do the high-res segmentation.

The final resolution of the end product will be determined by the resolution of the MNI template.

In this pipeline the Full CRUISE cortex extraction module will also create 3D volume level sets necessary for the surface inflation and mapping (central level set) and for layering (white matter-grey matter level set and grey matter-CSF level set).


Manual sanity checks and inspection after:

Skull stripping
T1 map before (`T1_map*_transform.nii`) vs. T1 map after (`T1_map*_strip.nii`) skull stripping

Segmentation
T1 map before (`T1_map*_bound.nii`) vs. T1 map after (`T1_map*_seg.nii`) segmentation



The `Surface mesh inflation` will create the surface (`*.vtk`) files (normal and inflated): they are based on the surface that goes across the middle of the cortical ribbon.

The `Surface mesh mapping` maps the data from the T1 map volume to the central surface. It can also be used to map a ROI defining volume onto a surface in case we want to only extract certain vertices from that surface.


## Other important modules
`Volumetric layering` module defines X layers using the WM-GM and GM-CSF 3D volumes level sets from the CRUISE module to create:
* a 4D volume of defining X+1 surfaces using level sets (`_surf.nii`) with X+1 volume in the 4th dimension
* a 3D volume layer label defining volume (`_labels.nii`) with X+1 labels (0 is for anything that is non-cortex).

The `Profile sampling` will sample the values of an intensity image (like the beta image of a GLM) to create the profiles along the X+1 surfaces defined by the 4D level set files of the layering.

The Surface mesh mapping can then be used to map those different surfaces by projecting them onto a reference surface (usually the central surface from the segmentation pipeline). The resulting `*.vtk` files has X+1 surfaces.

`Smooth surface` mesh data will smooth the data along a surface (`*.vtk`) with a given FWHM.

`Level set to mesh` converts a level set to a .vtk surface (no inflation).

`Embedded Syn` does non linear co-registration and can for example be used to correct for motion X inhomogeneity interactions.
It requires to install the [Advanced Normalization Tools](http://stnava.github.io/ANTs/) and will only work on linux as it calls some bash scripts (see the `layer-fmri-analysis.Layout` for a very simple example)

`Multimodal surface registration` for surface based registration

`Mesh data to volume` to map a surface onto a volume: needs a reference volume for dimensions, also make sure that the surface you input in non-inflated. Necessary to define in a volume a ROI drawn on a surface.
