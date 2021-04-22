# ALOS2_batch_processing
**This repo is used to process ALOS-2 time-series interferograms.**

**Note: some scripts could be found in $YOURPATH/GMTSAR/bin** 

---

### Step 0: add current cshell path to `~/.bashrc` or `~/.cshrc`.

### Step 1: convert the original data (CEOS format) to SLC and LED formats:
```shell
pre_proc_ALOS2_batch.csh  LED.list  batch.config  [n1]  [n2]
# n1 and n2 represent the range of ALOS-2 subswaths to process (n2 >= n1)
# Run from top-level processing directory
# LED.list should be in /raw, 
# batch.config should parallel to /raw, /topo, etc.

# This will generate a top-level /SLC directory which contains all of the pre-processed data
# Run time: ~XX min per scene
```

At this time, you may want to check the ```baseline_table.dat``` file to identify the supermaster scene. In this case, you should editmove the selected scene to the first line of the ```LED.list``` and```align.in``` files as well as update your ```batch.config``` file and then repeat Step 1 before proceeding.

### Step 2: align all slave images to the supermaster image.
```shell
align_ALOS2_swath_new.csh  align.in  n_swath  batch.config
# The first line of align.in should be the supermaster file
# n_swath is the ALOS-2 subswath to align (must do one at a time)
# align.in should parallel to /raw, /topo, etc.

# This will generate a new /F[1-5] directory for the subswath being processed. 
# It will be populated with another /SLC directory which will contain the aligned SLCs 

```
**In order to merge each subswath using the "merge_batch.csh" later on,
we upsample each SLC file to enforce each subswath to have the same
range sampling rate and PRF (azimuth) (using "samp_slc.csh").**


### Step 3: generate "topo_ra.grd" and "trans.dat" for each subswath.
``` shell
dem2topo_ra_swath.csh  n_swath  batch.config
```
### Step 4: select interferogram pairs.
```
get_baseline.py prm_file baseline_file

INPUT:
  prm_file - parameter (PRM) file containing date and baseline values for interferogram selection
  baseline_file - GMTSAR baseline table file
    
OUTPUT:
  short.dat - list of interferograms in YYYYMMDD_YYYYMMDD format
    Ex: 20150126_20150607')
        20150126_20150701')
        20150126_20150725')
        20150126_20150818')

  intf.in - list of interferogram pairs in SLC naming convention, for input into GMTSAR interferogram scripts
    Ex: S1A20150126_ALL_F1:S1A20150607_ALL_F1
        S1A20150126_ALL_F1:S1A20150701_ALL_F1

  Note: subsets of these will be generated which correspond to the selection parameters provided in the prm_file
    Ex:
    intf.in.sequential for SEQUENTIAL = True
    intf.in.skip_2 for Skip = 2
    intf.in.y2y for Y2Y_INTFS  = True

  baseline_plot.eps - plot of interferograms satisfying baseline constraints
```

### Step 5: make pairs of interferograms between any two pairs.
```shell
intf_ALOS2_batch_firkin.csh  intf.in  batch.config  swath  Ncores
```
NOTE: Do not try to process batch jobs for different swaths at the same time! They both rely on the same ```intf_alos.cmd``` file. Process one swath at a time (increase the number of cores to speed up).



### Optional: cut interferograms
Depending on study region, you may also want to cut interferograms at this stage. By default, GMTSAR will generate full-swath interferograms and only apply    ```region_cut``` at the unwrapping stage. To cut grd files en-masse after formation, you can use:
```
Usage: batch_cut.csh intf_list file_type new_file region

intf_list  - list of interferogram directories
    e.g.
    date1_date2
    date2_date3
    date3_date4
    ......
intf_dir   - path to directory containing interferograms
file_type  - filestem of product to cut (e.g. phase, phasefilt, corr)
new_file   - filestem for cut grids
region     - x_min/x_max/y_min/y_max (e.g. 0/10000/20000/40000
```

### Step 6: merge the filtered phase, correlation and mask grid files
```shell
merge_swath_ALOS2_list.csh  dir.list  dates.run  batch.config
# this command generate the necessary file list that will be used in merge_batch.csh
# dir.list in the form of directory of each subswath:
# ../F1/intf
# ../F2/intf
# ../F3/intf
# ...
# dates.run are date pairs between reference and repeat interferograms.

merge_batch_five.csh  inputfile  batch.config file_type
# input file is generated from executing merge_swath_ALOS2_list.csh
# file_type is the file stem of the grd file to be merged (e.g. phasefilt, phaseraw)
# the merging step would also generate merged "trans.dat" to be used in geocoding.
```
**Merging subswaths of ALOS-2 is similar to merging those of Sentinel-1.
You need to run "merge_swath" twice. To merge the topo_ra.grd, you need to
consider two extra factors:**
1. gmt FLIPUD each topo_ra.grd of each subswath (Because SLC indexs from upper left).
2. subtract the difference of Earth radius of each subswath.

### Step 7: unwrap each interferogram and geocode them.
```shell
unwrap_parallel.csh  dates.run  threshold  Ncores
# threshold means the coherence threshold of SNAPHU unwrapping 

proj_ra2ll.csh  trans.dat  phase.grd  phase_ll.grd
# phase.grd: an input GRD file using radar coordinates
# phase_ll.grd: an output GRD file using geographic coordinates.
```
