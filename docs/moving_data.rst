Moving Data
-----------

The main way to use ufs2arco is by creating a yaml "recipe" file, which
describes:

|

.. admonition::
        **mover**: whether or not to use Message Passing Interface (MPI) to parallelize data transfer
   :collapsible: closed

    + **name**: *str* 
        Specifies whether or not to use Message Passing Interface (MPI)
         Available Options:
          * 'datamover': does not use MPI, moves data serially
          * 'mpidatamover': uses MPI, allows for parallelization of data migration
    + **batch_size**: *int*
        Batch size; Must be >=1
         For 'datamover', determines how many chunks to split the data into for serial data transfer.
         For 'mpidatamover', determines how many processes will run in parallel to transfer data.

.. admonition::
        **directories**: directories specifying where to store the results
   :collapsible: closed

    + **zarr**: *str* 
        Filename for zarr output after ufs2arco run on source dataset 
         *Example: 'dataset.zarr'*
    + **cache**: *str*
    + **logs**: *str*

.. admonition::
        **source**: the data source
   :collapsible: closed

    + **name**: *str* 
        Specifies which source dataset to pull
         Available Options:
          * 'aws_aorc': NOAA's Analysis of Record for Calibration (AORC) dataset hosted on an AWS S3 bucket
          * 'conus404': NCAR's CONterminous United States for over 40 years at 4 km resolution (CONUS404) dataset hosted on Planetary Computer's Azure Blob Storage 
          * 'gcs_era5_1degree': European Centre for Medium-Range Weather Forecasts (ECMWF) Re-Analysis 5th Generation (ERA5) dataset hosted on NOAA's Unified Forecast System (UFS) Google Cloud Storage (GCS) bucket
          * ':ref:`aws_gefs_archive<gefs-archive>`': GEFS Archive
          * ':ref:`gfs_archive<gfs-archive>`': GFS Archive
          * ':ref:`aws_hrrr_archive<hrrr-archive>`': NOAA's High-Resolution Rapid Refresh (HRRR) Archive dataset hosted on an AWS S3 bucket
          * 'gcs_replay_atmosphere': Replay
    + **time**: 
          * **start**: *datetime*
              Start date and time of dataset to pull
               Example: '2017-01-01T00'
          * **end**: *datetime*
              End date and time of dataset to pull
               Example: '2017-01-01T18'
          * **freq**: *datetime*
              Interval between each time step pulled
               Example: '6h'
    + **variables**: *list* 
        Specifies which 
    + **levels**: *list* 
        Specifies which 
    + **slices**: *, optional* 
        Specifies which   
          * **sel**
    + **uri**: *str, optional* 
        Specifies which 

.. admonition::
        **target**: target data format
   :collapsible: closed

    + **name**: *str* 
        Specifies which target format to save the data in
         Available Options:
          * 'base': 
          * ':ref:`anemoi<anemoi-target>`': 
    + **sort_channels_by_levels**: *boolean*
    + **forcings**: *list*
        Specifies which target format to save the data in
         Available Options:
          * o
              Start da
          * o
              End da
          * o
              Interval between
    + **chunks**: *list*
        Specifies which target format to save the data in
         Available Options:
          * o
              Start da

.. admonition::
        **transforms**: *optional*, transforms to apply to the data
   :collapsible: closed

    + **divide**: *str* 
        Specifies which target format to save the data in
         Available Options:
          * 'base': 
          * ':ref:`anemoi<anemoi-target>`': 
    + **mappings**: *boolean*
    + **horizontal_regrid**: *list*
    + **rotate_vectors**
    + **vertical_regrid**


|

Once that recipe is created, the following command is used to run the workflow
serially::

    ufs2arco recipe.yaml   

In order to use MPI to parallelize the data transfer, for example over 64
processes, one would use::

    mpirun -n 64 ufs2arco recipe.yaml

Note that some machines may require different commands here.
For example
`NERSC's Perlmutter <https://docs.nersc.gov/systems/perlmutter/architecture/>`_
requires users to use ``srun`` not ``mpirun``.
Note also that the yaml recipe was inspired by
`anemoi-datasets <https://anemoi.readthedocs.io/projects/datasets/en/latest/>`_
in spirit, but the actual format and capabilities are a bit different.

This page is still a work in progress, and will describe the nuts and bolts of moving data.
In the meantime, feel free to raise an issue on the repo with questions, and
check out example recipe files in the
`ufs2arco integration tests directory
<https://github.com/NOAA-PSL/ufs2arco/tree/main/tests/integration>`_
for some examples to help you get started.


Missing Data
############

There are a number of reasons for data to be missing from the target dataset.

1. The file could be missing from the data source

2. The file could be corrupted 

3. The transfer could have failed for some reason (e.g., local disk failure,
   node failure, SLURM timeout, etc) 

When ufs2arco cannot find data for any of these reasons, it produces a warning and moves
on.
Missing data samples can be found in the following ways:

1. Look for a yaml file written in the same directory as the target zarr store,
   prefixed with ``missing`` and suffixed with ``.yaml``.
   This will list all the missing dates, forecast hours, ensemble members, etc.

2. Check the result dataset attributes for ``missing_data`` (base target) or
   ``missing_dates`` (anemoi target). Note that for anemoi datasets only keeps
   track of missing dates, so we could have 100 ensemble members present for one
   date, but if one ensemble member is missing, that date is ignored.

2. Look for the following warning in the root logfile (this example comes from
   preparing :ref:`GFS data<gfs-archive>`)::

    [1306 s] [WARNING] ⚠️  Some data are missing.
    [1306 s] [WARNING] ⚠️  The missing dimension combos, i.e., ('t0', 'fhr')
    [1306 s] [WARNING] ⚠️  were written to: /pscratch/sd/t/timothys/nested-eagle/v0/data/missing.gfs.analysis.zarr.yaml
    [1306 s] [WARNING] You can try running
    [1306 s] [WARNING]      python -c 'import ufs2arco; ufs2arco.Driver(''/path/to/your/original/recipe.yaml'').patch()'
    [1306 s] [WARNING] to try getting those data again

   Note that this shows where the yaml file noted above is located.

3. Run ``grep -A 1 WARNING`` inside of the log directory::

    log.0000.0256.out:[1306 s] [WARNING] ⚠️  Some data are missing.
    log.0000.0256.out:[1306 s] [WARNING] ⚠️  The missing dimension combos, i.e., ('t0', 'fhr')
    log.0000.0256.out:[1306 s] [WARNING] ⚠️  were written to: /pscratch/sd/t/timothys/nested-eagle/v0/data/missing.gfs.analysis.zarr.yaml
    log.0000.0256.out:[1306 s] [WARNING] You can try running
    log.0000.0256.out:[1306 s] [WARNING]    python -c 'import ufs2arco; ufs2arco.Driver(''/path/to/your/original/recipe.yaml'').patch()'
    log.0000.0256.out:[1306 s] [WARNING] to try getting those data again
    log.0000.0256.out-
    --
    log.0067.0256.out:[730 s] [WARNING] GFSArchive: Trouble finding the file: filecache::s3://noaa-gfs-bdp-pds/gfs.20210202/00/gfs.t00z.pgrb2.0p25.f000
    log.0067.0256.out-      dims = {'t0': Timestamp('2021-02-02 00:00:00'), 'fhr': np.int64(0)}, file_suffix = 
    --
    log.0112.0256.out:[176 s] [WARNING] GFSArchive: Could not find sp, will stop reading variables for this sample
    log.0112.0256.out-      dims = {'t0': Timestamp('2016-01-15 06:00:00'), 'fhr': np.int64(0)}, file_suffixes = ['']

   Note that this shows two missing data instances: one where the file couldn't be
   found (i.e., on 2021-02-02T00 forecast hour (fhr) 0), and one where the file was found,
   but it's corrupted (could not find sp).
