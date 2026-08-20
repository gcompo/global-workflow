.. _experiment-setup:

================
Experiment Setup
================

 Global workflow uses a set of scripts to help configure and set up the drivers (also referred to as Workflow Manager) that run the end-to-end system. While currently we use a `ROCOTO <https://github.com/christopherwharrop/rocoto/wiki/documentation>`__ based system and that is documented here, an `ecFlow <https://www.ecmwf.int/en/learning/training/introduction-ecmwf-job-scheduler-ecflow>`__ based systm is also under development and will be introduced to the Global Workflow when it is mature. To run the setup scripts, you need to have rocoto and a python3 environment with several specific libraries. The easiest way to guarantee this is to source the following script, which will load the necessary modules for your machine:

 ::

   source dev/ush/gw_setup.sh

.. warning::
   Sourcing gw_setup.sh will wipe your existing lmod environment

.. note::
   Bash shell is required to source gw_setup.sh

Scripts that will be used:

   * ``dev/workflow/setup_expt.py``
   * ``dev/workflow/setup_workflow.py``

.. note::
   **NOAA-PSL coupled reanalysis experiments:** ``./dev/workflow/create_experiment.py``
   is the primary supported setup path.  This script consumes a case YAML file and
   performs both experiment-directory configuration and workflow generation by calling
   the underlying ``setup_expt.py`` and ``setup_workflow.py`` scripts.  Direct use of
   those lower-level scripts remains available for debugging, development, and
   advanced/manual workflows, but ``create_experiment.py`` is the recommended entry
   point.  See `NOAA-PSL Coupled Reanalysis Setup`_ for a complete example.

****************************************
Step 1: Set user settings
****************************************

To make it easy for a user to specify the some of the user specific variables, users can create a ``.gwrc`` file in their home directory.  An example is provided in ``$TOP_OF_CLONE/dev/parm/workflow/gwrc`` that contains the following variables:

   - ACCOUNT: the account to charge for the run (Slurm charging account)
   - PATH_ACCOUNT: the filesystem project account used to construct paths such as ``HOMEDIR``, ``STMP``, and ``PTMP``.  On most platforms this defaults to ``ACCOUNT``.  On Gaea C6, the Slurm charging account and the ``/gpfs/f6/`` filesystem project directory may differ, so ``PATH_ACCOUNT`` can be set independently of ``ACCOUNT``.
   - HOMEDIR: the home directory of the user
   - STMP: the path to the DATAROOT storage area for the run
   - PTMP: the path to the COMROOT storage area for the run
   - HPSS_PROJECT: the project on HPSS to charge for the run

This file is read by the ``setup_expt.py`` script to set the user specific variables. If you do not have a ``.gwrc`` file, the setup script will revert to the default values in the repository.

.. note::
   **Gaea C6 users:** If your Slurm charging account differs from the ``/gpfs/f6/`` filesystem project directory, set both ``ACCOUNT`` (Slurm) and ``PATH_ACCOUNT`` (filesystem) in your ``.gwrc``.  For example::

      user:
        ACCOUNT: 'new-charging-account'
        PATH_ACCOUNT: 'ira-da'

   ``PATH_ACCOUNT`` can also be set when using ``create_experiment.py`` by specifying a custom ``gwrc`` path in the experiment YAML's ``experiment`` section::

      experiment:
        gwrc: '/path/to/my_custom.gwrc'
        ...

   where ``my_custom.gwrc`` contains the ``PATH_ACCOUNT`` entry shown above.

***************************************
Step 2: Run experiment generator script
***************************************

The following command examples include variables for reference but users should not use environmental variables but explicit values to submit the commands. Exporting variables like EXPDIR to your environment causes an error when the python scripts run. Please explicitly include the argument inputs when running both setup scripts:

::

   cd dev/workflow

   ./setup_expt.py NET MODE --idate $IDATE --edate $EDATE [--app $APP] [--start $START]
     [--interval $INTERVAL_GFS] [--resdetatmos $RESDETATMOS] [--resdetocean $RESDETOCEAN]
     [--pslot $PSLOT] [--configdir $CONFIGDIR] [--comroot $COMROOT] [--expdir $EXPDIR] [--gwrc $GWRC]
     [--resensatmos $RESENSATMOS] [--nens $NENS] [--run $RUN]

where:

   * ``NET`` is the first positional argument that initializes the parser for the correct system.  Valid values are:
       - gfs: Global Forecast System (GFS)
       - gefs: Global Ensemble Forecast System (GEFS)
       - sfs: Seasonal Forecast System (SFS)
       - gcafs: Global Chemistry and Aerosol Forecast System (GCAFS)
   * ``MODE`` is the second positional argument that instructs the setup script to produce an experiment directory for mode of execution.  Valid options are:
       - forecast-only: for running a forecast only experiment
       - cycled: for running a cycled experiment (forecast + data assimilation)

   Based on the ``NET`` and ``MODE`` arguments, the setup script will provide further input options to the user. The script will also check for the existence of the ``$ROTDIR`` and ``$EXPDIR`` directories and prompt the user to overwrite them if they already exist.

   * ``$APP`` is the target application, one of:

     - ATM: atmosphere-only [default]
     - ATMA: atm-aerosols
     - ATMW: atm-wave (currently non-functional)
     - S2S: atm-ocean-ice
     - S2SA: atm-ocean-ice-aerosols
     - S2SW: atm-ocean-ice-wave
     - S2SWA: atm-ocean-ice-wave-aerosols

   * ``$START`` is the start type (``warm`` or ``cold`` [default: ``cold``])
   * ``$IDATE`` is the initial start date of your run (first cycle date in ``YYYYMMDDCC``)
   * ``$EDATE`` is the ending date of your run (``YYYYMMDDCC``) and is the last cycle that will complete [default: ``$IDATE``]
   * ``$PSLOT`` is the name of your experiment [default: ``test``]
   * ``$CONFIGDIR`` is the path to the ``/config`` folder under the copy of the system you're using [default: ``$TOP_OF_CLONE/dev/parm/config/$NET``]
   * ``$RESDETATMOS`` is the resolution of the atmosphere component of the system (i.e. 768 for C768) [default: ``384``]
   * ``$RESDETOCEAN`` is the resolution of the ocean component of the system (i.e. 0.25 for 1/4 degree) [default: ``0.``; determined based on atmosphere resolution]
   * ``$INTERVAL_GFS`` is the forecast interval in hours [default: ``6``]
   * ``$COMROOT`` is the path to your experiment output directory. Your ``ROTDIR`` (rotating com directory) will be created using ``COMROOT`` and ``PSLOT``. [default: ``$HOME`` (but do not use default due to limited space in home directories normally, provide a path to a larger scratch space)]
   * ``$EXPDIR`` is the path to your experiment directory where your configs will be placed and where you will find your workflow monitoring files (i.e. rocoto database and xml file). DO NOT include PSLOT folder at end of path, it will be built for you. [default: ``$HOME``]
   * ``$GWRC`` is the custom user global-workflow resource configuration file. [default: ``$HOME/.gwrc`` or ``$TOP_OF_CLONE/dev/parm/workflow/gwrc``]

   For the ``cycled`` mode, additional options are available:

   * ``$SDATE_GFS`` cycle to begin GFS forecast [default: ``$IDATE + 6``]
   * ``$RESENSATMOS`` is the resolution of the atmosphere component of the ensemble forecast [default: ``192``]
   * ``$NENS`` is the number of ensemble members [default: ``20``]
   * ``$RUN`` is the starting phase [default: ``gdas``]

Examples:

Forecast-only with Atm-only configuration in the GFS:

::

   cd dev/workflow
   ./setup_expt.py gfs forecast-only --pslot test --idate 2020010100 --edate 2020010118 --resdetatmos 384 --interval 6 --comroot /some_large_disk_area/Joe.Schmo/comroot --expdir /some_safe_disk_area/Joe.Schmo/expdir

Forecast-only with Coupled model configuration in the GFS:

::

   cd dev/workflow
   ./setup_expt.py gfs forecast-only --app S2SW --pslot coupled_test --idate 2013040100 --edate 2013040100 --resdetatmos 384 --comroot /some_large_disk_area/Joe.Schmo/comroot --expdir /some_safe_disk_area/Joe.Schmo/expdir

Forecast-only with the Coupled model (including aerosols) in the GFS:

::

   cd dev/workflow
   ./setup_expt.py gfs forecast-only --app S2SWA --pslot coupled_test --idate 2013040100 --edate 2013040100 --resdetatmos 384 --comroot /some_large_disk_area/Joe.Schmo/comroot --expdir /some_safe_disk_area/Joe.Schmo/expdir

Cycled with the Atmosphere-only model (including ensembles) in the GFS:

::

   cd dev/workflow
   ./setup_expt.py gfs cycled --app ATM --pslot cycled_test --idate 2013040100 --edate 2013040100 --comroot /some_large_disk_area/Joe.Schmo/comroot --expdir /some_safe_disk_area/Joe.Schmo/expdir --resdetatmos 384 --resensatmos 192 --nens 80 --interval 6

******************************************
Step 3: Check user and experiment settings
******************************************

Go to your ``EXPDIR`` and check the following variables within your ``config.base`` now before running the next script:

   * ``ACCOUNT``
   * ``PATH_ACCOUNT`` (filesystem project account; defaults to ``ACCOUNT`` on most platforms; see note in Step 1 for Gaea C6 users)
   * ``HOMEDIR``
   * ``STMP``
   * ``PTMP``
   * ``ARCDIR`` (location on disk for online archive used by verification system)
   * ``HPSSARCH`` (YES turns on archival)
   * ``HPSS_PROJECT`` (project on HPSS if archiving)
   * ``ATARDIR`` (location on HPSS if archiving)

Some of those variables will be found within a machine-specific if-block so make sure to change the correct ones for the machine you'll be running on.

`NOTE`: If you selected ``ARCHCOM_TO='globus_hpss``, then you will need to activate your globus connections between Mercury and MSU.  See :doc: globus_arch.rst for more details.

Now is also the time to change any other variables/settings you wish to change in ``config.base`` or other configs. `Do that now.` Once done making changes to the configs in your EXPDIR go back to your clone to run the second setup script. See :doc:configure.rst for more information on configuring your run.
Go to your ``EXPDIR`` and check/change the following variables within your ``config.base`` now before running the next script.

*************************************
Step 4: Run workflow generator script
*************************************

This step sets up the files needed by the Workflow Manager/Driver. At this moment only Rocoto configurations are generated:

::

   ./setup_workflow.py $EXPDIR/$PSLOT rocoto|ecflow

Example:

::

   ./setup_workflow.py /some_safe_disk_area/Joe.Schmo/expdir/test rocoto

Additional options for setting up Rocoto or ecFlow are available with `setup_workflow.py -h` that allow users to change the number of failed tries, number of concurrent cycles and tasks as well as verbosity levels.

Presently, only the Rocoto workflow engine is supported.  EcFlow capabilities are a work in progress.

****************************************
Step 5: Confirm files from setup scripts
****************************************

You will now have a rocoto xml file in your ``$EXPDIR`` (``$PSLOT.xml``) and a crontab file generated for your use. Rocoto uses CRON as the scheduler. If you do not have a crontab file you may not have had the rocoto module loaded. To fix this load a rocoto module and then rerun setup_workflow.py script again. Follow directions for setting up the rocoto cron on the platform the experiment is going to run on.

.. _noaa-psl-coupled-reanalysis-setup:

*******************************************
NOAA-PSL Coupled Reanalysis Setup
*******************************************

For NOAA-PSL coupled reanalysis experiments, ``./dev/workflow/create_experiment.py``
is the primary supported setup path.  This script consumes a case YAML file and
performs both experiment-directory configuration and workflow generation by invoking
``setup_expt.py`` and ``setup_workflow.py`` internally.  Direct use of those
lower-level scripts may remain available for debugging, development, and
advanced/manual workflows.

On Gaea C6 the Slurm charging account (``HPC_ACCOUNT`` → ``ACCOUNT``) and the
``/gpfs/f6/`` filesystem project directory (``PATH_ACCOUNT``) may differ.  Both
can be supplied as environment variables on the command line.  When they are the
same (most platforms), only ``HPC_ACCOUNT`` is needed; ``PATH_ACCOUNT`` will
default to ``ACCOUNT``.

**Gaea C6 — filesystem project and Slurm account differ:**

::

   HPC_ACCOUNT="<slurm-account>" PATH_ACCOUNT="<filesystem-project-account>" \
   pslot="<experiment-name>" \
   RUNTESTS="/gpfs/f6/<filesystem-project-account>/scratch/$USER/GWTESTS" \
   ICSDIR_ROOT="/gpfs/f6/<filesystem-project-account>/proj-shared/<path-to-ics>" \
     ./dev/workflow/create_experiment.py --yaml dev/ci/cases/coupledreanl/C192mx025_3DVarAOWCDA.yaml

Concrete Gaea C6 example (``ira-da`` project, filesystem project and Slurm account are the same):

::

   HPC_ACCOUNT="ira-da" \
   pslot="C192coupled3dvar_test" \
   RUNTESTS="/gpfs/f6/ira-da/scratch/$USER/GWTESTS" \
   ICSDIR_ROOT="/gpfs/f6/ira-da/proj-shared/Jeffrey.S.Whitaker/replayics/C192mx025" \
     ./dev/workflow/create_experiment.py --yaml dev/ci/cases/coupledreanl/C192mx025_3DVarAOWCDA.yaml

**Orion/Hercules (** ``PATH_ACCOUNT`` **defaults to** ``HPC_ACCOUNT`` **):**

::

   HPC_ACCOUNT="gsienkf" \
   pslot="C192coupled3dvar_test" \
   RUNTESTS="/work2/noaa/gsienkf/$USER/GWTESTS" \
   ICSDIR_ROOT="/work/noaa/gsienkf/whitaker/replayics/C192mx025" \
     ./dev/workflow/create_experiment.py --yaml dev/ci/cases/coupledreanl/C192mx025_3DVarAOWCDA.yaml

Alternatively, ``PATH_ACCOUNT`` (and other user settings such as ``ACCOUNT``) can
be placed in a custom ``gwrc`` file and referenced via the experiment YAML's
``experiment`` section:

::

   experiment:
     gwrc: '/path/to/my_custom.gwrc'
     ...

where ``my_custom.gwrc`` contains:

::

   user:
     ACCOUNT: 'new-charging-account'
     PATH_ACCOUNT: 'ira-da'

After ``create_experiment.py`` completes, the experiment is ready to run:

::

   cd $EXPDIR/$PSLOT
   rocotorun -w $PSLOT.xml -d $PSLOT.db
   scrontab $PSLOT.crontab
   ln -fs $HOMEGFS/dev/workflow/rocoto_viewer.py .
   ./rocoto_viewer.py -w $PSLOT.xml -d $PSLOT.db   # monitor progress

.. note::
   Log files are written to ``$RUNTESTS/COMROOT/$PSLOT/logs``.  If
   ``rocoto_viewer`` shows a workflow step failed, check the log file in that
   directory for the error message.
