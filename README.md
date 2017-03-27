# Suite2p: fast, accurate and complete two-photon pipeline#

Registration, cell detection, spike extraction and visualization GUI. 

Details in [http://biorxiv.org/content/early/2016/06/30/061507](http://biorxiv.org/content/early/2016/06/30/061507).

[![IMG](https://img.youtube.com/vi/xr-flH2Ow2Y/0.jpg)](https://www.youtube.com/watch?v=xr-flH2Ow2Y)

This code was written by Marius Pachitariu and members of the lab of Kenneth Harris and Matteo Carandini. It is provided here with no warranty. For support, please open an issue directly on github. 

### I. Introduction ###

This is a complete, automated pipeline for processing two-photon Calcium imaging recordings. It is simple, fast and yields a large set of active ROIs. A GUI further provides point-and-click capabilities for refining the results in minutes. The pipeline includes the following steps


1) X-Y subpixel registration --- using a version of the phase correlation algorithm and subpixel translation in the FFT domain. If a GPU is available, this completes in 20 minutes per 1h of recordings at 30Hz and 512x512 resolution.

2) SVD decomposition --- this provides the input to cell detection and accelerates the algorithm. 

3) Cell detection --- using clustering methods in a low-dimensional space. The clustering algorithm provides a positive mask for each ROI identified, and allows for overlaps between masks. 

4) Signal extraction --- by default, all overlapping pixels are discarded when computing the signal inside each ROI, to avoid using "demixing" approaches, which can be biased. The neuropil signal is also computed independently for each ROI, as a weighted pixel average, pooling from a large area around each ROI, but excluding all pixels assigned to ROIs during cell detection. The neuropil subtraction coefficient is estimated by maximizing the skewness of F - coef*Fneu (F is the ROI signal, Fneu is the neuropil signal). The user is encouraged to also try varying this coefficient, to make sure that any scientific results do not depend crucially on it. 

5) Manual curation --- the output of the cell detection algorithm can be visualized and further refined using the included GUI. The GUI is designed to make cell sorting a fun and enjoyable experience. It also includes an automatic classifier that gradually refines itself based on the manual labelling provided by the user. This allows the automated classifier to adapt for different types of data, acquired under different conditions. 

6) Spike deconvolution --- cell and neuropil traces are further processed to obtain an estimate of spike times and spike "amplitudes". The amplitudes are proportional to the number of spikes in a burst/bin. Even under low SNR conditions, where transients might be hard to identify, the deconvolution is still useful for temporally-localizing responses. The cell traces are corrected using the ICA-derived neuropil contamination coefficients, and baselined using the minimum of the overly-smoothed trace. 

### II. Getting started ###

The toolbox runs in Matlab (+ one mex file) and currently only supports tiff file inputs. To begin using the toolbox, you will need to make local copies (in a separate folder) of two included files: master_file and make_db. It is important that you make local copies of these files, otherwise updating the repository will overwrite them (and you can lose your files). The make_db file assembles a database of experiments that you would like to be processed in batch. It also adds per-session specific information that the algorithm requires such as the number of imaged planes and channels. The master_file sets general processing options that are applied to all sessions included in make_db, UNLESS the option is over-ridden in the make_db file.  The global and session-specific options are described in detail below. 

For spike deconvolution, you need to run mex -largeArrayDims SpikeDetection/deconvL0.c (or .cpp under Linux/Mac)

Below we describe the outputs of the pipeline first, and then describe the options for setting it up, and customizing it. Importantly, almost all options have pre-specified defaults. Any options specified in master_file in ops0 overrides these defaults. Furthermore, any option specified in the make_db file (experiment specific) overrides both the defaults and the options set in master_file. This allows for flexibility in processing different experiments with different options. The only critical option that you'll need to set is ops0.diameter, or db(N).diameter. This gives the algorithm the scale of the recording, and the size of ROIs you are trying to extract. We recommend as a first run to try the pipeline after setting the diameter option. Depending on the results, you can come back and try changing some of the other options.  

Note: some of the options are not specified in either the example master_file or the example make_db file. These are usually more specialized features.

### III. Outputs. ###

The output is a struct called dat which is saved into a mat file in ResultsSavePath using the same subfolder structure, under a name formatted like F_M150329_MP009_2015-04-29_plane1. It contains all the information collected throughout the processing, and contains the ROI and neuropil traces in Fcell and FcellNeu, and whether each ROI j is a cell or not in stat(j).iscell. stat(j) contains information about each ROI j and can be used to recover the corresponding pixels for each ROI in stat.ipix. The centroid of the ROI is specified in stat as well. Here is a summary of where the important results are:

cell traces are in dat.Fcell
neuropil traces are in dat.FcellNeu
manual, GUI overwritten "iscell" labels are in dat.cl.iscell
 
stat(icell) contains all other information:
iscell: automated label, based on anatomy
neuropilCoefficient: neuropil subtraction coefficient, based on maximizing the skewness of the corrected trace (ICA)
st: are the deconvolved spike times (in frames)
c:  are the deconvolved amplitudes
kernel: is the estimated kernel


### III. Input-output file paths ###

RootStorage --- the root location where the raw tiff files are  stored.

RegFileRoot --- location on local disk where to keep the registered movies in binary format. This will be loaded several times so it should ideally be an SSD drive. 

ResultsSavePath --- where to save the final results. 

DeleteBin --- deletes the binary file created to store the registered movies

RegFileTiffLocation --- where to save registered tiffs (if empty, does not save)

All of these filepaths are completed with separate subfolders per animal and experiment, specified in the make_db file. Your data MUST be stored under a file tree of the form

\RootStorage\mouse_name\session\block\*.tif(f)

### IV. Options for registration ###

showTargetRegistration --- whether to show an image of the target frame immediately after it is computed. 

PhaseCorrelation --- whether to use phase correlation (the alternative is normal cross-correlation).

SubPixel --- accuracy level of subpixel registration (10 = 0.1 pixel accuracy)

kriging --- compute shifts using kernel regression with a gaussian kernel of width 1 onto a grid of 1/SubPixel

NimgFirstRegistration --- number of randomly sampled images to do the target computation from

NiterPrealign --- number of iterations for the target computation (iterative re-alignment of subset of frames)

smooth_time_space --- convolves raw movie with a Gaussian of specified size in specified dimensions;
                      [t]: convolve in time with gauss. of std t, [t s]: convolve in time and space,
                      [t x y]: convolve in time, and in space with an ellipse rather than circle
                      
++ Block Registration (for high zoom/npixels - assumes scanning is in Y direction) ++

nonrigid --- set to 1 for non-rigid registration (or set numBlocks > 1)

numBlocks --- number of blocks dividing y-dimension of image (default is 6)

blockFrac --- percent of image to use per block (default is 1/(numBlocks-1))
             
quadBlocks --- interpolate block shifts to single line shifts (6 blocks -> 512 lines) by fitting a quadratic function (default is 1)

smoothBlocks --- if quadBlocks = 0, then smoothBlocks is the standard deviation of the gaussian smoothing kernel

++ Bidirectional scanning issues (frilly cells) taken care of automatically ++

### V. Options for cell detection ###

clustModel --- what clustering model to use; "neuropil" is described in the paper, adds a neuropil contribution to each pixel individually. "standard" is plain clustering. 

neuropilSub --- for neuropil subtraction. "none" does not output a neuropil trace; "surround" estimates the signal in an annulus around the cell after ignoring all ROIs detected during optimization (even non-cells); "model" is described in the paper, uses spatial basis functions to estimate a smoothly-varying neuropil signal over the FOV and regresses this out simultaneously with the estimation of ROI responses. 

sig --- spatial smoothing constant: smooths out the SVDs spatially. Makes ROIs more cell-shaped. 

Nk0 --- starting the algorithm with this many clusters

Nk --- final annealed number of clusters (warning, due to the structure of the annealing process, one should have Nk0<3*Nk or else the code crashes. No need to make it larger anyway). 

nSVDforROI --- how many SVD components to keep for clustering. Usually ~ the number of expected cells (Nk). 

ShowCellMap --- whether to show the clustering results as an image every 10 iterations of the clustering

niterclustering --- how many iterations of clustering

getROIs --- whether to run the ROI detection algorithm after registration

### Options for SVD decomposition ###

NavgFramesSVD --- for SVD, data has to be temporally binned. This number specifies the final number of points to be obtained after binning. In other words, datasets with many timepoints are binned in higher windows while small dataset are binned less. 

getSVDcomps --- whether to obtain and save to disk SVD components of the registered movies. Useful for pixel-level analysis and for checking the quality of the registration (residual motion will show up as SVD components). This is a separate SVD decomposition from that done for cell clustering (does not remove a running baseline of each pixel). 

nSVD --- how many SVD components to keep.

### VII. Options for spike deconvolution ###

imageRate --- imaging rate per plane. Approximate, for initialization of deconvolution kernel.  

sensorTau --- decay half-life (or timescale). Approximate, for initialization of deconvolution kernel.

maxNeurop --- neuropil contamination coef has to be less than this (sometimes good, i.e. for interneurons)

### VIII. Rules for post-clustering ROI classification ###

These options serve to compute candidate cell clusters, that can then be refined in the GUI. Clusters computed in the algorithm are split into connected regions and then classified as cell/non-cell based on the following

diameter --- % expected diameter of cells (used for 0.25 * pi/4*diam^2 < npixels < 10*pi/4*diam^2). Automatically sets MinNpix and MaxNpix below (1/4). 

MaxNpix --- maximum number of pixels per ROI. Automatically set by diameter, if provided. 

MinNpix --- minimum number of pixels per ROI. Automatically set by diameter, if provided. 


### IX. Example database entry ###

The following is a typical database entry in the local make_db file, which can be modelled after make_db_example. The folder structure assumed is RootStorage/mouse_name/date/expts(k) for all entries in expts(k). 

i = i+1;
db(i).mouse_name    = 'M150329_MP009'; 
db(i).date          = '2015-04-27';
db(i).expts         = [5 6]; % which experiments to process together

Other (hidden) options are described in make_db_example.m, and at the top of run_pipeline.m (set to reasonable defaults), and get_signals_and_neuropil.m (neuropil "surround" option). 



